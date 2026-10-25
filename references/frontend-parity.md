# 脱敏迁移经验：前端一致性（样式与行为一样）

本记录来自两次 PHP 传统 Web 应用向 C#/.NET 迁移的案例。结论：后端接口通了只是半成品，用户认的是“样式前端啥的也得一样”——URL、页面结构、样式、表单行为必须逐项对齐。

## 1. 静态资源：原样搬运，不重写

- 把原 `view/` 下的 `css/js/img/fonts` 按原相对路径拷入 `wwwroot/`（如 `wwwroot/view/...`），页面引用路径保持不动。
- 第三方库（bootstrap/jquery/md5 等）能本地化的就本地化进 `wwwroot/lib`，不要把原来能离线跑的站改成强依赖 CDN。
- 上传目录（附件/头像）映射为静态文件或专用控制器，原 URL 前缀（如 `upload/`）保持可访问。

## 2. 模板：逐页转 Razor，只换数据标签

- 原 `view/htm/*.htm` 与 Razor 页面一一对应，先建空页面占位，再逐页搬 HTML 骨架。
- 只替换数据输出点：`<?php echo $x;?>` → `@Model.X`；循环 `foreach` → `@foreach`；条件 `if` → `@if`。class、id、内联样式、JS 一个字符都不要“顺手优化”。
- 布局页（header/footer/导航/面包屑）先还原，再做内容页；导航里的用户信息（登录态显示）与后端双轨登录联调。
- 分页条、时间格式（相对时间/绝对时间）、转义规则（昵称转义、帖子内容富文本不过滤）与原站一致，否则老用户一眼看出不对。

## 3. 表单与 JS 行为：字段名和变换一个不能变

- 表单 `name`（如 `email/username/password/subject/message/code`）与后端 `param()` 名严格一致，改名即断。
- 前端变换必须保留：如登录/注册页先 `md5(密码)` 再 POST，后端校验公式依赖这个输入形态；验证码、 synlogin/token 透传等流程同理。
- ajax 返回体沿用原协议（如 `{code, message, ...}`，`code` 可能是字段名也可能是数字），前端 JS 按原逻辑判断，不要改成 REST 风格后让老 JS 失效。
- 富文本/ubb/markdown 渲染与 XSS 过滤是显示契约：同一份内容在新旧站渲染结果要一致，至少抽样帖子逐字节比对。B类（WordPress 迁移经验：`the_content` + `kses`）V1 可用 `HtmlEncode + 段落/换行还原` 先对齐结构，shortcode/oembed/wpautop 列入“已知差异”。另见验证篇：含中文帖子页必须原样输出中文字符、无 `&#x;` 实体（Razor 默认编码器会转义 CJK，需放开）。

## 4. URL 与跳转：外观不变

- 旧 URL 双模式（`?x-1.htm` / `x-1.htm`）继续可用；页面内链接生成函数统一走一处（对标原 `url()`），rewrite 开关只影响生成不影响解析。
- 登录/退出/发帖后的跳转目标（referer 回跳）与原站一致。
- SEO 相关（title/keywords/description、版块 seo 字段）继续输出，老收录不受影响。

## 5. Razor 特有的三个坑（与 PHP 原样输出对齐）

- **HtmlEncoder 默认转义 CJK**：Razor 默认把中文转成 `&#x;` 实体，而 PHP 是原样 UTF-8 输出。必须放开（如 `TextEncoderSettings(UnicodeRanges.All)`），否则全站中文全是实体，肉眼立即可见差异。
- **回帖 return_html**：原站快速回帖要求服务端直接返回渲染好的 `<li>` HTML 并插入列表。做法是把列表 partial 抽出来，服务端用视图渲染引擎（如 `IRazorViewEngine`）渲染成字符串返回，保持原 JS 一字不改。
- **modal 表单页无布局**：后台 modal（如置顶/移动/删除）加载的是完整 HTML 片段，C# 侧对应视图设无布局，只输出 card + `ajax-eval` 脚本。
- **Razor 属性表达式**：`href="...-@x.Id.htm"` 会被解析成成员访问，必须写成 `@(x.Id)`；条件 attribute 用 `Html.Raw` 拼整段标签。
- **表单脚本必须走节输出**：Razor 布局的库脚本（jQuery/xiuno.js）在 body 尾部加载，页面内联 `<script>` 会先于 `$` 执行而整体失效——表现就是拦截器没挂上，原生提交把 JSON 直接显示出来（某次迁移发帖页真实踩坑）。表单 partial 只放 HTML，JS 拆到独立 partial，由调用页 `@section Scripts` 引入；用 node `--check` 对渲染出的脚本做语法校验，用“绑定脚本位置 > jQuery 位置”做顺序断言。

## 6. 插件移植（Rank Math SEO 迁移经验：只迁前端契约 + 存储键，不迁后台 JS/ scoring）

插件体量大（1669 文件）时不要全量移植，按“前端可观察输出 → 存储键 → 后台最小闭环”切三刀：

- **head 输出顺序照抄**：`<title>` → description → robots → canonical（noindex 时去掉 canonical + prev/next）→ OpenGraph → Twitter → JSON-LD → 站长验证 → credit 注释收尾。标题/描述/robots/canonical 逐个对齐回退链：文章 meta 覆盖 → 摘要 → 类型模板（`%title% %sep% %sitename%` / `%excerpt%`）→ 首段截断 160。
- **变量模板只实现用到的**：`%title%/%sep%/%sitename%/%sitedesc%/%excerpt%/%date%/%search_query%/%page%/%category%/%author%`，未知变量吞掉并压空格，不要原样输出 `%xxx%`。
- **JSON-LD 图谱最小闭环**：WebSite（`@id` 必须用站点首页，不用当前页 URL）+ WebPage + Article（仅文章页：headline/datePublished/dateModified/author/publisher）+ BreadcrumbList（面包屑 HTML 同步输出一份 JSON-LD）。Twitter 卡片按有无图片切 `summary` / `summary_large_image`。
- **存储键原名复用**：文章 SEO 覆盖存 `rank_math_title/description/canonical_url/robots/focus_keyword` 等 postmeta 键，选项存 `titles_*` 模板 + `general_*` 验证 + `breadcrumbs_enable`，旧库直挂即兼容。
- **重定向独立中间件**：精确路径匹配（大小写不敏感、去尾斜杠）+ 301/302/307/410/451 + 命中计数，放在美化路由中间件之前。
- **明确不迁的**（写进“已知差异”）：SEO 打分/内容分析、AI/统计/外链检查等纯后台 JS 模块、sitemap 图片扩展、多尺寸附件。

## 7. 可复用的前端核对表

- 静态资源路径逐个 200；离线可跑（无必需外网依赖）。
- 模板页清单 vs 原 `view/htm/` 清单，无遗漏页面。
- 每个表单的字段名、前端变换、返回体 code 语义与原 JS 一致。
- 新旧站同帖渲染结果抽样比对一致。
- 首页/版块/主题/登录/注册/个人中心六大页截图并排比对，结构样式无肉眼差异。
