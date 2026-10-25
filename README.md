# php-to-csharp

把传统 PHP Web 应用完整迁移到 C#（ASP.NET Core + FreeSql + MariaDB/MySQL），
保外部可观察行为：URL、密码/token、数据、前端样式，再选 .NET 实现。

Migrate legacy PHP web apps to C# (ASP.NET Core + FreeSql + MariaDB/MySQL),
preserving externally observable behavior — URLs, passwords/tokens, data, frontend —
then choose the .NET implementation.

## 适用类型 / Project types（先判定，再选模板，不要混用 / classify first, never mix）

- **A类 / Type A**：单文件入口 + 查询串路由、MySQL 直写 SQL、无框架模板视图、
  cookie + session 双轨登录、前端预哈希（`md5(明文)` 再 POST）、对称加密 token。
  Single-entry + query-string routing, raw MySQL, framework-less views,
  cookie + session dual login, client-side pre-hash, symmetric-encrypted tokens.
- **B类 / Type B（WordPress 迁移经验 / WordPress migration experience）**：
  明文 POST、多轨密码校验（裸哈希→可移植哈希→现代哈希）、HMAC cookie + 服务端 token 表、
  朴素 query + 美化路由双模式。
  Plain-text POST, multi-track password verification (raw → portable → modern hashes),
  HMAC cookie + server-side token table, plain-query + pretty-permalink dual routing.

## WordPress 迁移与兼容性 / WordPress migration & compatibility

已用本 skill 完成 WordPress → C# 全栈迁移（对照案例），兼容点如下。
Completed a full WordPress → C# migration with this skill; compatibility points:

- **表结构 / Schema**：12 表全量建模（`wp_posts/postmeta/users/usermeta/terms/
  term_taxonomy/relationships/termmeta/comments/commentmeta/options/links`），
  表名/字段/主键与原表一致，旧库直挂可读；字符集 `utf8mb4`。
  All 12 tables modeled 1:1 (names/fields/keys); legacy DB mounts read-only OK.
- **密码 / Passwords**：裸哈希→phpass 可移植哈希→bcrypt 现代哈希三轨验
  （对标 `wp_check_password`，含 `$wp` 的 SHA-384 预哈希），新用户默认可移植哈希，
  与 PHP 双向互验。
  Three-track verification (raw → phpass portable → bcrypt modern, incl. `$wp` SHA-384 pre-hash).
- **登录态 / Sessions**：`login|exp|token|HMAC-SHA256` cookie（对标
  `wp_generate_auth_cookie`，含 passfrag 与 scheme 盐规则）+ 服务端 token 表回查，
  八盐键须与 `wp-config.php` 一致，否则存量 cookie 失效。
  HMAC cookies + server-side token lookup; all eight salt keys must match `wp-config.php`.
- **路由 / Routing**：朴素 `?p=/?page_id=/?s=/?cat=` 与美化
  `/年/月/日/slug/`、分类/标签/日期/作者/附件双模式恒解析，生成走重写开关；
  中间件必须放行 MVC 首段（大小写敏感）与静态资源，且位于 `UseRouting` 之前。
  Dual plain/pretty routing; middleware allow-lists MVC segments (case-sensitive) before `UseRouting`.
- **内容 / Content**：`the_content` 回退链（meta→摘要→模板→截断）、短代码剥离、
  `wpautop` 段落化、评论 `kses` 过滤；Razor 默认转义 CJK 为 `&#x;` 实体，
  必须放开到全部 Unicode 区间（原版输出原样 UTF-8）。
  Content fallback chain, shortcode stripping, wpautop, comment filtering;
  relax Razor's encoder to full Unicode so CJK stays raw UTF-8.
- **订阅抓取 / Feeds**：RSS2/Atom/评论 Feed、Sitemap（含文章/页面拆分）、robots（含自定义规则）。
- **接口与后台 / API & admin**：`/wp-json/wp/v2` 只读（posts/pages/media/comments/
  users/categories/tags/search/types/statuses）+ 评论创建；后台最小闭环
  （仪表盘/文章/页面/评论审核/用户/设置/上传）；原版路径全兼容
  （`wp-login.php`、`wp-comments-post.php`、`wp-admin`、`xmlrpc` 标准 fault）。
- **插件 / Plugins**：Rank Math SEO 已按“前端契约 + 存储键”移植
  （head 全套标签、JSON-LD 图谱、面包屑、重定向 301/302/307/410/451、后台 SEO 页），
  详见 [插件化经验](references/plugin-system.md) 与 [前端一致性§6](references/frontend-parity.md)。

## 完成态 / Done criteria（Xiuno + WordPress 双案例 / two cases）

前台 + 后台 + 插件及范例已全部移植，老数据库直挂、老 URL 全兼容，
自检页全绿 + 端到端回归全过，可无缝替换原版。
Frontend + admin + plugins fully ported; legacy DB mounts, legacy URLs all work;
compat page green + end-to-end regression green — drop-in replaceable.
判定“可替换”的门槛：自检全绿 + 回归全过 + 旧库挂载验证通过，三者缺一不可。
Replaceable = green compat + green regression + mounted-legacy-DB verification. All three required.

## 快速使用 / Quick use

```text
使用 $php-to-csharp 梳理并实现这项 PHP 到 C# 迁移，优先保留现有行为契约。
Use $php-to-csharp to plan and implement this PHP-to-C# migration,
preserving existing behavior contracts first.
```

或调用 skill 工具载入 `php-to-csharp`（完整指令见 [SKILL.md](SKILL.md)），
IDE 侧默认提示词见 [agents/openai.yaml](agents/openai.yaml)。
Or load `php-to-csharp` via the skill tool (full instructions in [SKILL.md](SKILL.md));
IDE default prompt in [agents/openai.yaml](agents/openai.yaml).

## 工作流程（摘要，细则见 SKILL.md）/ Workflow (summary; details in SKILL.md)

1. 盘点行为契约（表结构、密码链、token/cookie、路由、模板变量），不猜测。
   Inventory behavior contracts (schema, password chain, tokens/cookies, routing, template vars) — no guessing.
2. 建四层骨架：Core（纯函数）/ Data（建表 + 种子）/ Services（逐函数移植）/ Web（中间件 + 控制器 + Razor）。
   Four-layer skeleton: Core (pure functions) / Data (tables + seeds) / Services (per-function port) / Web (middleware + controllers + Razor).
3. 先过兼容自检（`/Compat` 式自检页），全绿再写业务。
   Pass the compat self-check page first; only then build features.
4. 双模式路由兼容，中间件放 `UseRouting` 之前，且放行 MVC 首段与静态资源。
   Dual-mode routing; middleware before `UseRouting`, allow-listing MVC segments and static assets.
5. 前端 1:1 还原（资源原搬、模板只换数据标签、表单 JS 不变）；插件按“前端契约 + 存储键”移植。
   Pixel/behavior frontend parity; port plugins by frontend contract + storage keys.
6. 端到端收尾，输出“已验证等价 / 已知差异 / 未覆盖”三类清单。
   Finish end-to-end; report verified-equivalent / known-diffs / uncovered.

## 文档索引 / Docs

| 文档 / Doc | 内容 / Covers |
|---|---|
| [SKILL.md](SKILL.md) | 完整指令：工作流程 + 核心提醒 + 全部分支入口 / Full instructions, workflow, reminders, branch index |
| [references/php-web-compat.md](references/php-web-compat.md) | 密码/token/URL/session 字节级兼容，含 A类与 WordPress 多轨/HMAC 两套模板 / byte-level compat, Type A + WordPress templates |
| [references/data-layer-migration.md](references/data-layer-migration.md) | 建表、种子、连带计数、旧库直挂 / tables, seeds, cascading counts, legacy-DB mounting |
| [references/frontend-parity.md](references/frontend-parity.md) | 模板、静态资源、表单 JS、Razor 特有坑、插件移植 / templates, assets, form JS, Razor pitfalls, plugin ports |
| [references/plugin-system.md](references/plugin-system.md) | 通用插件机制：Hook 总线 + 模块装配 + UI 扩展点 / plugin system: hook bus, modules, UI zones |
| [references/plugin-sdk.md](references/plugin-sdk.md) | 插件 SDK 与老插件迁移，含 PHP→C# 映射表 / plugin SDK migration incl. PHP→C# mapping |
| [references/verification.md](references/verification.md) | 自检页、端到端脚本、收尾核对表 / compat page, e2e scripts, sign-off checklist |
| [agents/openai.yaml](agents/openai.yaml) | IDE 默认提示词 / IDE default prompt |
