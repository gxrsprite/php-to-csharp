# 脱敏迁移经验：数据层迁移

本记录来自两次 PHP 传统 Web 应用向 C#/.NET（FreeSql + MariaDB）迁移的案例，只保留通用工程经验。已去除项目名、真实库名、真实账号与部署信息。

## 1. 安装描述文件就是建表规范

表结构描述文件（含建表 + 默认组/默认版/管理员种子）是唯一可信的表契约：

- 表名含前缀的，C# 实体用 `[Table(Name = "xxx")]` 逐表锁定，不要依赖 FreeSql 默认命名。
- 字段名、键（主键/联合主键/普通索引）与 PHP 一致，保证旧库直挂可读。
- 无符号语义注意：`int unsigned` 的 IP/时间字段用 `uint` 存，`tinyint` 状态位用最小够用的整数类型并在赋值处显式转换，避免隐式转换编译错。
- 自增主键标 `[Column(IsIdentity = true, IsPrimary = true)]`，联合主键逐个标 `IsPrimary`。

## 2. 列名必须逐字对齐，不能靠默认映射

这是最容易翻车的点：ORM 默认用属性名当列名（如 `MessageFmt`），而 PHP 原库是全小写（`message_fmt`），多词也不一定按下划线拆（`firstpid` 不是 `first_pid`，`userip` 不是 `user_ip`）。

- 每个属性显式标注列名（如 FreeSql `[Column(Name = "...")]`），表名同样显式。
- 上线前写脚本逐表逐列比对：解析安装 SQL 的字段清单，与 `INFORMATION_SCHEMA.COLUMNS` 做**大小写敏感**对比（很多比较默认不区分大小写，会漏掉 PascalCase 问题）。
- 发现不一致时重建库验证，不要在旧库上猜。

## 3. 字符集与引擎：只升不降

- 旧库字符集（如 `utf8`）→ 新库用其超集（如 `utf8mb4`），连接串显式声明，老数据照读、新表情可存。
- 旧引擎（如 MyISAM）→ FreeSql CodeFirst 新建表为 InnoDB，只影响新库；旧库直挂时不要反向改引擎。
- CodeFirst 只负责“新库建表”，旧库迁移走 SQL 导入，不在启动时对旧表做结构同步。

## 4. 种子数据与安装 SQL 等价

`DbInit`-style 种子只在空库时写入，内容与表结构描述文件逐项对应：

- 用户组行（含各组权限位）缺一不可，游客组/管理组缺失会导致权限过滤全站放行或全站拦截。
- 管理员种子行（含密码 hash 与 salt）原样保留，默认口令是什么就写什么并在文档里写明默认账号口令。
- 默认版块/默认配置行同样处理。

## 5. 连带计数是移植主体，不是附带（A类示例，B类同理换成文章/评论/术语计数）

`thread_create/post_delete/user_delete` 这类函数的主体工作是连带更新：

- 建主题：先建首贴 → 建主题 → 回填 `tid` → 用户主题数/版块主题数/全站统计/mythread/附件关联，失败要回滚已建的首贴。
- 回帖：更新主题 last 信息与回帖数 → 用户回帖数 → mythread/mypost → 版块今日数/全站统计。
- 删主题/删用户：级联删帖子/附件/统计/关联表，并回减计数。

移植时按原函数逐行列出副作用清单，一条一条在 C# 里落实并用端到端验证（见 [验证经验](verification.md)），不要只建主表。

## 6. 配置映射

原站默认配置文件（A类 `conf.default.php`／B类 `wp-config.php`）→ `appsettings.json` 的旧站配置节：DB（host/port/user/password/name/charset）、站点名、分页大小、功能开关（注册/找回/重写）、盐密钥（WordPress 迁移经验：八键 `AUTH_KEY/SECURE_AUTH_KEY/LOGGED_IN_KEY/NONCE_KEY + 四 SALT`）。
连接串模板示例（占位符，勿填真实值）：

```text
server=<db-host>;port=<db-port>;user=<db-user>;password=<db-password>;database=<db-name>;charset=utf8mb4;SslMode=none
```

WordPress 迁移经验补充：表结构描述文件 12 表（`wp_posts/postmeta/users/usermeta/terms/term_taxonomy/relationships/termmeta/comments/commentmeta/options/links`，单站不含多站表）；选项表核心种子（`siteurl/home/blogname/template/stylesheet/permalink_structure/posts_per_page/users_can_register/default_role`）；登录 token 存用户元数据表（`session_tokens`）；评论计数由新增评论函数回写；术语计数同理。新 token 存 JSON 时验签用子串匹配兼容旧序列化老行。

## 7. 可复用的数据核对表

- 全部表（以实际表结构描述文件为准）是否全量建模，有无遗漏会话扩展表/队列表/用户 token 行这类“隐形表”。
- 主键/自增/联合键是否与原库一致；旧库直挂后自增值是否连续。
- 种子行数与描述文件等价（A类组/版/管理员，B类默认管理员 + 示例文章 + 示例页面 + 默认分类）；默认管理员能否用默认口令登录。
- 每条 create/delete 的连带计数是否有端到端断言覆盖。
- 字符集/排序规则在连接串、建表、旧库三处是否一致。
