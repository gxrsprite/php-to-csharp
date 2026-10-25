# php-to-csharp

[English](README.md)

把传统 PHP Web 应用完整迁移到 C#（ASP.NET Core + FreeSql + MariaDB/MySQL），
保外部可观察行为：URL、密码/token、数据、前端样式，再选 .NET 实现。

## 适用类型（先判定，再选模板，不要混用）

- **A类**：单文件入口 + 查询串路由、MySQL 直写 SQL、无框架模板视图、
  cookie + session 双轨登录、前端预哈希（`md5(明文)` 再 POST）、对称加密 token。
- **B类（WordPress 迁移经验）**：明文 POST、多轨密码校验（裸哈希→可移植哈希→现代哈希）、
  HMAC cookie + 服务端 token 表、朴素 query + 美化路由双模式。

## WordPress 迁移与兼容性

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

## 完成态（Xiuno + WordPress 双案例）

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
5. 前端 1:1 还原（资源原搬、模板只换数据标签、表单 JS 不变）；插件按“前端契约 + 存储键”移植。
6. 端到端收尾，输出“已验证等价 / 已知差异 / 未覆盖”三类清单。

## 文档索引

| 文档 | 内容 |
|---|---|
| [SKILL.md](SKILL.md) | 完整指令：工作流程 + 核心提醒 + 全部分支入口 |
| [references/php-web-compat.md](references/php-web-compat.md) | 密码/token/URL/session 字节级兼容（含可编译代码块） |
| [references/data-layer-migration.md](references/data-layer-migration.md) | 建表、种子、连带计数、旧库直挂 |
| [references/frontend-parity.md](references/frontend-parity.md) | 模板、静态资源、表单 JS、Razor 特有坑、插件移植 |
| [references/plugin-system.md](references/plugin-system.md) | 通用插件机制经验 |
| [references/plugin-sdk.md](references/plugin-sdk.md) | 插件 SDK 与老插件迁移（含 PHP→C# 映射表） |
| [references/verification.md](references/verification.md) | 自检页、端到端脚本、收尾核对表 |
| [agents/openai.yaml](agents/openai.yaml) | IDE 默认提示词（display_name / short_description / default_prompt） |
