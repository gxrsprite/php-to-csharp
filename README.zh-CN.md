# php-to-csharp

[English](README.md)

把传统 PHP Web 应用完整迁移到 C#（ASP.NET Core + FreeSql + MariaDB/MySQL），
保外部可观察行为：URL、密码/token、数据、前端样式，再选 .NET 实现。

## 适用类型（先判定，再选模板，不要混用）

- **A类**：单文件入口 + 查询串路由、MySQL 直写 SQL、无框架模板视图、
  cookie + session 双轨登录、前端预哈希（`md5(明文)` 再 POST）、对称加密 token。
- **B类（WordPress 迁移经验）**：明文 POST、多轨密码校验（裸哈希→可移植哈希→现代哈希）、
  HMAC cookie + 服务端 token 表、朴素 query + 美化路由双模式。

---

## 第一块：WordPress 迁移（B类）

已用本 skill 完成 WordPress → C# 全栈迁移（对照案例），兼容点如下：

- **表结构**：12 表全量建模（`wp_posts/postmeta/users/usermeta/terms/
  term_taxonomy/relationships/termmeta/comments/commentmeta/options/links`），
  表名/字段/主键与原表一致，旧库直挂可读；字符集 `utf8mb4`。
- **密码**：裸哈希→phpass 可移植哈希→bcrypt 现代哈希三轨验
  （对标 `wp_check_password`，含 `$wp` 的 SHA-384 预哈希），新用户默认可移植哈希，
  与 PHP 双向互验。
- **登录态**：`login|exp|token|HMAC-SHA256` cookie（对标
  `wp_generate_auth_cookie`，含 passfrag 与 scheme 盐规则）+ 服务端 token 表回查，
  八盐键须与 `wp-config.php` 一致，否则存量 cookie 失效。
- **路由**：朴素 `?p=/?page_id=/?s=/?cat=` 与美化
  `/年/月/日/slug/`、分类/标签/日期/作者/附件双模式恒解析，生成走重写开关；
  中间件必须放行 MVC 首段（大小写敏感）与静态资源，且位于 `UseRouting` 之前。
- **内容**：`the_content` 回退链（meta→摘要→模板→截断）、短代码剥离、
  `wpautop` 段落化、评论 `kses` 过滤；Razor 默认把 CJK 转成 `&#x;` 实体，
  必须放开到全部 Unicode 区间（原版输出原样 UTF-8）。
- **订阅抓取**：RSS2/Atom/评论 Feed、Sitemap（含文章/页面拆分）、robots（含自定义规则）。
- **接口与后台**：`/wp-json/wp/v2` 只读（posts/pages/media/comments/
  users/categories/tags/search/types/statuses）+ 评论创建；后台最小闭环
  （仪表盘/文章/页面/评论审核/用户/设置/上传）；原版路径全兼容
  （`wp-login.php`、`wp-comments-post.php`、`wp-admin`、`xmlrpc` 标准 fault）。
- **插件**：Rank Math SEO 已按“前端契约 + 存储键”移植
  （head 全套标签、JSON-LD 图谱、面包屑、重定向 301/302/307/410/451、后台 SEO 页），
  详见 [插件化经验](references/plugin-system.md) 与 [前端一致性§6](references/frontend-parity.md)。

---

## 第二块：Xiuno 论坛迁移（A类）

已用本 skill 完成 Xiuno BBS 4.0 → C# 全栈迁移，机制与覆盖如下：

- **密码链**：前端先 `md5(明文)` 再 POST，后端只做 `md5(hex+salt)`；
  安装脚本看似双哈希，与上式是同一个（它拿到的是明文）。默认管理员 `admin / 1` 照原库可登。
- **登录态**：session uid + `bbs_token` 双轨，
  token 明文 `"$ip\t$time\t$uid\tmd5(password)"` 经 XXTEA（小端 + int32 语义）加密 +
  base64 + 自定义 urlencode，key 为 `md5(auth_key)`，超 30 分钟回查密码；
  后台另有独立 `bbs_admin_token`（明文 `"$ip\t$time"`，
  key=`md5(md5(useragent)+auth_key)`，3600 秒有效、1800 秒滑动续发）。
- **路由**：`?thread-1.htm` 与 `thread-create-1.htm` 双模式、`admin/?forum-list.htm`
  等后台链接全部改写到同一控制器；中间件位于 `UseRouting` 之前，
  精确路径分支必须让位给旧形态解析（`/admin/` 吞查询串会导致登录页自环 302）。
- **数据**：17 张表名列名与安装 SQL 逐字一致（大小写敏感比对脚本验证，
  ORM 默认 PascalCase 列名是头号坑）；字符集用超集 `utf8mb4`；
  `thread_create/post_delete/user_delete` 的连带计数逐函数落实。
- **前端**：静态资源原样搬运、模板逐页转 Razor（只换数据标签）、
  表单字段名与 JS 行为不变、ajax `{code,message}` 协议原样返回；
  发帖表单 JS 必须走节输出（内联会先于 jQuery 执行导致原生提交把 JSON 显示出来）；
  快速回帖 `return_html` 由服务端渲染 `<li>` HTML，原 JS 一字不改。
- **覆盖**：前台 13 类页、版主置顶/移动/删除/关闭、附件上传/删除/下载、
  后台 9 模块（登录/首页/版块/用户组/用户/主题搜索/设置/SMTP/缓存/插件管理）。

### 插件如何兼容（本案例重点）

PHP 插件就三样东西，对应三种搬法，**没有自动移植**：

| PHP 侧 | C# 侧 | 成本 |
|---|---|---|
| `conf.json` 清单 | 文件夹插件 `plugin.json`（字段相同）/ 代码插件接口属性 | 照抄 |
| `hook/*.htm` 纯 HTML 片段 | 原样放入 `hook/`，挂载点名不变，视图加一行 `@Html.PluginSlot("point")` | **零成本，直接复用**（`my_hello` 范例） |
| 静态文件 | 启动同步到站点 `wwwroot/plugins/<dir>/`，URL `/plugins/<dir>/...` | 照搬（`xn_scroll_top` 范例，JS 原样复用） |
| `hook/*.php` | 按挂载点语义重写：模板点→片段/Razor，逻辑点→服务方法/控制器动作 | 逐个手工重写 |
| `install.php` 纯 SQL 部分 | `install.sql`，后台一键执行；业务部分→ `InstallAsync` | 低 |
| `route/*.php` | 控制器动作或 `MapEndpoints` 的 Minimal API | 中 |
| `setting.php + setting.htm` | 插件自带设置页（如 `/plugin/<dir>/setting`，kv 存取，键名原样保留） | 中（`xn_insert_code` 范例：7 键 7 槽同构迁移） |
| `model/*.php` | Services 方法（含连带计数一并搬） | 中 |
| `overwrite/` 整文件覆盖 | **不提供等价机制**，逐个重写并记为已知差异 | 高 |

配套设施（`src/XiunoBBS.Sdk` 独立包 + 后台统一管理页）：

- `IXiunoPlugin`：`Dir/Name/Brief/Version` + `ConfigureServices / MapEndpoints /`
  `RenderSlot / InstallAsync / UninstallAsync / SettingUrl`（后四个有默认实现），注释自带上表。
- `CodePluginHost`：反射发现 + 启动显式装入 `Xiuno.Plugin.*.dll`
  （程序集懒加载，不预装就扫不到）。
- `PluginStateService`：代码插件启停 + 插件设置存 `bbs_kv`。
- 后台插件页：文件夹/代码统一列表，启用/禁用/安装表/设置入口。
- 挂载点只加不改、命名沿用原站 hook 名（如 `body_start`、`footer_end`、
  `index_main_start`、`thread_user_after`）。
- 信任模型与 PHP 一致：仅管理员可安装/启用插件，片段原样输出。

---

## 完成态（双案例共同标准）

前台 + 后台 + 插件及范例已全部移植，老数据库直挂、老 URL 全兼容，
自检页全绿 + 端到端回归全过，可无缝替换原版。
判定“可替换”的门槛：自检全绿 + 回归全过 + 旧库挂载验证通过，三者缺一不可。

## 快速使用

```text
使用 $php-to-csharp 梳理并实现这项 PHP 到 C# 迁移，优先保留现有行为契约。
```

或调用 skill 工具载入 `php-to-csharp`（完整指令见 [SKILL.md](SKILL.md)），
IDE 侧默认提示词见 [agents/openai.yaml](agents/openai.yaml)。

## 工作流程（摘要，细则见 SKILL.md）

1. 盘点行为契约（表结构、密码链、token/cookie、路由、模板变量），不猜测。
2. 建四层骨架：Core（纯函数）/ Data（建表 + 种子）/ Services（逐函数移植）/ Web（中间件 + 控制器 + Razor）。
3. 先过兼容自检（`/Compat` 式自检页），全绿再写业务。
4. 双模式路由兼容，中间件放 `UseRouting` 之前，且放行 MVC 首段与静态资源。
5. 前端 1:1 还原（资源原搬、模板只换数据标签、表单 JS 不变）；插件按映射表迁移。
6. 端到端收尾，输出“已验证等价 / 已知差异 / 未覆盖”三类清单。

## 文档索引

| 文档 | 内容 |
|---|---|
| [SKILL.md](SKILL.md) | 完整指令：工作流程 + 核心提醒 + 全部分支入口 |
| [references/php-web-compat.md](references/php-web-compat.md) | 密码/token/URL/session 字节级兼容（含可编译代码块，A/B 双轨） |
| [references/data-layer-migration.md](references/data-layer-migration.md) | 建表、种子、连带计数、列名对齐、旧库直挂 |
| [references/frontend-parity.md](references/frontend-parity.md) | 模板、静态资源、表单 JS、Razor 特有坑 |
| [references/plugin-system.md](references/plugin-system.md) | 通用插件机制经验（B类/Rank Math 验证） |
| [references/plugin-sdk.md](references/plugin-sdk.md) | 插件 SDK 与老插件迁移（Xiuno 迁移总结，含 PHP→C# 映射表） |
| [references/verification.md](references/verification.md) | 自检页、端到端脚本、收尾核对表 |
| [agents/openai.yaml](agents/openai.yaml) | IDE 默认提示词（display_name / short_description / default_prompt） |
