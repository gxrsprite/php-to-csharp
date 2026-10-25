# Discuz! X3.5 / X5 首批迁移经验

本篇来自用户提供的 X3.5 SC UTF-8 20260504 与 X5.0.4.Open 20261010 源码包。当前验证范围是版本盘点、契约/实体、X3.5 首批普通论坛、后台版块新增/详细设置和配置复制/合并/删除；其余后台、插件、完整界面及真实生产库副本验证尚未完成。以下不包含真实凭据、密钥或部署路径。

## 选择源码和迁移顺序

- 固定用户 ZIP 的 SHA-256 与源码 version/release，不用网上同名分支覆盖。两个版本分别保留源码、契约和实体命名空间。
- 本次提取安装结构（含 UC）为 X3.5 290 表/2747 列、X5 300 表/2865 列；X5 新增 15 表、移除 5 表，88 个共有表列定义有变化。此计数只对应两份安装包，不能推广到插件/分表生产库。
- X3.5 传统模板、BBCode 更适合先迁；X5 的 MitFrame 分发、cells、多主题和 JSON 内容需独立实现。X5 帖子额外有 `content/source` JSON、`original`，不要只输出 message 后宣布兼容。
- X5 建表 SQL 藏在有 `<?php exit...?>` 保护头的 `install/sql/sql_install.php` / `sql_uc.php`；默认数据则为语言安装目录中的 `table_*.php` 数组，经 PHP 安装器序列化。不能用 X3.5 的 install_data.sql 填 X5。

## UC 密码与论坛 cookie 不是同一份密码

- 这两份包登录交给 UC，表单传明文；不要因旧公式也有双 MD5 就套 Xiuno 的前端预哈希。
- `ucenter_members.password/salt`：六位盐使用 `md5(md5(明文)+salt)`；空盐走 PHP password_verify（默认 bcrypt，也可能配置其他算法）。转换盐及 Argon2 应逐项支持或明确拒绝，不能误判通过。
- `common_member.password` 是论坛的独立随机凭据，不是可验证登录口令。cookie 明文为 `forumPassword + "\t" + uid`，经 authcode 编码；解码后必须回查论坛当前凭据和账号状态。
- 有效密钥 `md5(config.security.authkey + saltkeyCookie)`；authcode 内部还会再 MD5，不能省掉一层。基础 cookiepre 还会追加 `substr(md5(cookiepath + "|" + cookiedomain),0,4) + "_"`。
- `formhash` 使用时间戳字符串去掉最后七位，加 username、uid、有效 authkey；MD5 结果截取偏移 8 的 8 位。后台另加固定域字符串。
- 旧 authkey、基础 cookiepre、path/domain、saltkey 必须保留。前后台独立认证、安全提问、UC HTTP 同步通知都另有契约，不能以前台登录通过代替覆盖。

## 独立自增与联合主键

- UC 和论坛会员 uid 都是 AUTO_INCREMENT，但论坛必须显式插入 UC 返回 uid。FreeSql 普通 Insert 会忽略 identity 属性，即使对象里已赋值，需 `InsertIdentity()`。让两表自增序列产生间隔后注册，是有效的回归用例。
- `forum_post` 的主键是 `(tid, position)`，pid 是唯一键，来自 `forum_post_tableid` 的自增分配，不能把 pid 当普通 ORM identity。
- 并发回帖锁住 thread 行后推进 maxposition，插帖子与回复数/最后发表/用户和版块计数在同一事务中完成。只测串行回帖会漏掉 position 冲突。
- MyISAM 不具备相同事务回滚保证；普通线程/回帖计数通过也不代表积分、日志、缓存、通知、跨日统计、审核和父级统计已等价。
- `posttableid` 决定帖子分表后缀；生产库可能另有主题分表和归档会员。安装 SQL 表全集不等于现网表全集。

## 样式是带语义的输入

- 原 `common.css/module.css` 含 `{IMGDIR}/{BGCODE}` 等变量，需先替换为安装主题变量及派生 BGCODE，不能当成浏览器可直接加载的纯 CSS。
- module.css 的 `/** forum::index,group::index **/ ... /** end **/` 由 PHP `class_template::cssvtags` 按当前 app/module 筛选。整份载入会让后面的其他页面样式覆盖当前页面；本次浏览器检查实际发现并修复。
- 保留 class/id、原静态 URL、字体路径和版权标识。默认头像在 UC 的 data/avatar/images 路径，不能假设在 static/image/common 下；浏览器检查图片才能发现这类 404。
- 原表单 inajax 常走 XML/HTML/JS 回调，不能套 `{code,message}` JSON。原生 POST/302 可以作为首批闭环，但应列为明确的交互差异。
- 未迁移 `[hide]`、密码主题等权限语义时，不能原样显示其内部正文。基础 HTML 编码能阻止脚本，却不能阻止敏感内容泄露。

## 验证应能证明什么

- 调原 PHP 文件中的 authcode 函数，只替换 time/microtime 输入生成固定向量；对比 C# 编码字节及解码。中文、空串、真实 cookie 明文都纳入。
- 用 PHP 原函数解 C# cookie，再用 PHP 生成 cookie 交给 C#；C# 生成 bcrypt 再由 PHP password_verify。这样避免“自己写错又自己验证对”。
- SQL 契约核对用大小写敏感列名、顺序、可空性、自增、主键声明顺序。主键顺序应单独保存，不能根据列出现顺序推断联合主键顺序。
- FreeSql 的默认映射优先级是 AOP < FluentApi < Attribute：实体已有 `[Table(Name=...)]` 时，AOP/ConfigEntity 配置前缀看起来执行了也可能完全无效。本次自定义前缀测试实际抓到此问题；用 `UseMappingPriority(Aop, Attribute, FluentApi)` 后通过 ConfigEntity 设置表名，并检查 GetTableByEntity 的 DbName。默认前缀测试不能证明此能力。
- SQL CLI 的 batch 模式会转义字段内 TAB；比较 authcode 明文时加 `--raw`，否则真实 TAB 和字面 `\\t` 让正确实现被误判。
- 首批 HTTP 断言应覆盖旧口令、注册扩展表、UID 序列不连续、并发 position、计数、查询串与重写 URL、CSRF 和游客拒绝。测试数据库与现用旧站副本要分别记录。
- 明确当前未迁移模块。没有真实旧库副本和原 PHP 页面逐页对照，就不能使用“全兼容”“可无缝替换”作为结论。

## X3.5 后台版块管理的补充验证

- 本包后台 `discuz_admincp` 沿用前台 auth cookie，再检查 `common_admincp_session`：uid/panel=1、errorcount=-1、1800 秒、IP。不要套 X5 的独立 adminauth 方案，也不要只看 adminid 就让前台登录直进后台。后台表单 hash 必须加 PHP 的固定 Admin Control Panel 域字符串，另调原函数生成向量。
- 创始人需匹配旧 founder 配置及 groupid/adminid。普通后台成员的 cpgroupid=0 在原版代表完全权限；其他组按 `common_admincp_perm` 授权，`customperm` 是 PHP 序列化的排除项，不能反过来当新增授权。写入需 `_allowpost`，`forums_admin` 与 `forums_edit` 不能互相替代。
- 原 `adminipaccess` 是 IP 前缀匹配，不是 CIDR；X3.5 原函数把 CRLF 分隔符转换为候选前缀。认证策略中还要区分全局强制安全提问和用户组 forcesecques，未迁移时拒绝，不能静默绕过。
- `admin.php?action=forums` 是页面；带 frames=yes 则是框架，iframe 再请求同一 action。不要把所有 admin.php 无条件重定向到框架而形成 iframe 自套。旧 edit&fid 深链接通过二次登录后应保留目标；当前模块以外的操作应明确拒绝。
- 原字段 `newcat[]/newcatorder[]`、`newforum[fup][]/neworder[fup][]`、`name[fid]/order[fid]` 在 ASP.NET FormCollection 中是含括号的字符串键和 StringValues；不要假设默认模型绑定能按 PHP 的嵌套数组自动还原。新增名称要区分 PHP 字节长度和 SQL 字符长度。
- 原动态 addrow/deleterow 通过固定父节点层级找到 table。迁 Razor 时保持 a→div→td→tr→tbody→table 结构才能原样复用；本次已用浏览器实际点新增并保存验证，不只测服务端 POST。
- 新分区/版块必须同事务写 forum_forum 和 forum_forumfield。C# 新对象数值零会覆盖数据库默认值，需对照 PHP 新建分支与 SQL 默认值明确设置 alloweditpost、allowglobalstick、allowfeed 等；版主继承另写 forum_moderator。列表改名/排序仅更新对应列，不回写整个含旧统计的对象。
- 本机 FreeSql 3.5.311 与 MySqlConnector/MariaDB 的组合中，forum_forum ENUM 参数插入 group 出现截断；同一对象经 ORM 的 NoneParameter 转义字面量可正常写入。本次采用局部处理，不改旧 ENUM、不手工拼接名称。该结论仅限已验证的组合，其他驱动版本应先复现再选方案。
- C# 直接读表可即时展示新板块，但切回 PHP 时仍需使 forums 缓存失效。默认只读配置应在后台保存服务再次检查，不能依靠禁用按钮；后台认证有效也不意味着允许写旧库。
- 初批验证范围：原 CSS 字节相等、后台二次登录、创始人/委派排除权限、过期/IP/未验证会话拒绝、分区与版块/子版块批量新增、扩展表/默认值/版主继承、名称排序、隐藏简介、前台立即发帖、只读拒绝。后续详细设置和删除/合并/复制见下一节；完整原版操作日志仍未迁移。

## X3.5 详细设置、复制、合并和删除

- 先读 `admincp_forums.php` 每个 operation 及 `function_delete.php` 的调用参数。原版“复制”是覆盖目标版块的选中配置，绝不是复制帖子；“删除版块”调用 `deletethread` 时默认 `membercount=false, credit=false`，不能自行回扣个人发帖数和积分。
- 表单标量多为列名加 `new`；六项组权限却是 `viewperm[]/postperm[]/...`，入库后才成为前后有 TAB 的字符串。嵌套分类、积分和附件限制需显式解析括号键，保留数字 ID。必须允许“取消所有勾选”与“旧表单没有提交此项”有不同含义，可为完整界面增加 presence marker，部分旧 POST 只更新提交的字段。
- `simple` 同时编码排序字段、升降序、子版块显示和简洁模式，不能当布尔值。特殊主题用位掩码，自动关闭用带符号天数。风格、域名、导航、别名、分类、附件限制及积分规则 fids 各有自己的关联表，不能全塞到 JSON 配置中。
- PHP serialize 字符串长度按 UTF-8 字节计数；字典数字键必须保留为 PHP 整数键。仅实现标量/数组并限制长度、深度和数量，拒绝对象及引用，不执行 PHP。验证须让真实 PHP `unserialize` 读取新写入的中文分类、公式和积分数组；自身往返不能证明兼容。
- 自定义公式应做语法解析而非仅替换敏感单词或用字符白名单；校验括号、操作数和原版变量/操作符。保留原表达式及 `getuserprofile`/`$memberformula` 形式的编译字符串供旧 PHP 读取，但不在 C# 执行 PHP。设置可保存不等于前台公式、勋章、密码、积分结算已迁移；未迁权限仍应拒绝访问。
- 复制使用配置白名单，禁止通过 `options[]` 注入 fid/fup/type/name/帖子统计等字段；只更新选中的列。目标身份、内容和关系保留；覆盖积分策略还需同步全局规则的 fids。拒绝自身和分区目标，整批操作用同一事务。
- 合并主题分类时不要直接照搬 PHP `array_merge`：数字键会从零重排，主题里的全局 typeid 引用因此失配。应按全局 ID 合并 types/icons/moderators，移动分类行并保持 ID，重算目标帖子统计及最后发表，清除源元数据/导航/域名和旧缓存。对原行为的这类修正要明确记录。
- 安装 SQL 只含基础表，旧库可能有 thread/post 数字后缀表。通过 INFORMATION_SCHEMA 和可信核心表名单发现，动态标识符必须同时校验前缀和数字后缀，值参数化。合并、删除、确认数量和重算计数都要覆盖分表；不要因基础表为空就认定版块为空。开发测试实际创建专用分表验证，成功后只移除自己创建的表。
- `common_moderate` 是 PHP 分发表面，并不存在同名真实表；thread/post 实际审核记录在各自 moderate 表，按 id 清理。删除还包括 tid/pid/rpid/fid 关联核心表、附件分表及 EXIF、收藏/动态/标签、淘帖集合统计、相册关联、版主/个人访问授权、积分规则 fids、域名/导航及缓存。插件删除钩子不能当作已自动迁移。
- 保留旧 URL，不必保留危险的 GET 写入：GET 预览源/目标和实际数量，POST 带后台 formhash、用户/操作/源目标绑定且有时效的签名凭据；执行时加行锁重新核对内容快照。确认后新增主题必须使旧凭据失效。后台 op 权限与 `_allowpost`、只读开关每次重新检查，不能仅隐藏按钮。
- 发帖/回帖与删除/合并必须按同一顺序先锁论坛行，再锁主题并重新验证位置与权限，避免确认后新增孤儿行或合并后回帖仍写旧 fid。此保证仅适用于遵守锁协议的写入者，不能让旧 PHP/插件同时写库后声称并发等价。
- 数据库事务不包括文件系统。已验证方案是先把本地附件、投票图片、封面及缩略图移入站点静态根目录和附件根之外的私有隔离区，保存清单，数据库失败时反向恢复。路径拒绝根外跳转及符号链接，远程附件/封面未实现时在任何删除前拒绝。用临时数据库触发器制造删除失败，验证主记录、关联行及文件均恢复；不能把这当成断电原子性，也不能把附件隔离当成整站内容恢复。
- 已验证的删除/合并要求相关核心表为 InnoDB。旧库存在 MyISAM 时明确拒绝并要求在副本转换验证，不静默改表，也不宣称事务可回滚。原版 AJAX 分批 GO/TRUE 协议、远程存储、第三方插件、生产副本和大数据量仍须单独迁移/验证。
- 后台八个标签复用原 CSS class/资源；除 HTTP 参数测试，还要在浏览器提交完整表单。隐藏标签的输入仍会提交，须检查没有清空旧嵌套设置。此次浏览器完整保存、真实 PHP 读取、配置复制、过期快照拒绝、分表合并/删除、核心关联清理、文件故障恢复及独立只读拒绝已验证。
