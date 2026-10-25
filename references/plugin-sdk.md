# 插件 SDK 与老插件迁移（Xiuno BBS 迁移经验总结）

> 本篇标注来源：Xiuno BBS 4.0（PHP）→ C#（.NET 10 + FreeSql + MariaDB）迁移中沉淀。
> 仍遵守工厂脱敏规则：以下均为机制与结构经验，不含真实路径、密钥、账号、部署信息；
> 引用的上游插件文件名（如 `my_hello`、`xn_insert_code`）来自公开仓库，仅作形态示例。

## 1. 先看清 PHP 插件的三类形态，再定搬法（没有自动移植）

从公开插件仓抽样确认，插件目录结构是统一的：

- `conf.json`：清单（name/brief/version/enabled）。
- `hook/<point>.htm`：纯 HTML 片段，对应原站模板里的 `<!--{hook xxx}-->` 注释点，原样可复用。
- `hook/*.php`：逻辑钩子（函数入口/路由/语言包挂载），无 PHP 运行时，必须逐个手工重写。
- `install.php/unstall.php`：多为纯 SQL（建表/删表/加列），可转成 `install.sql` 直接执行；含业务逻辑的需手工转 C#。
- `route/*.php`、`model/*.php`、`view/*.htm`、`setting.php/setting.htm`：分别对应控制器、服务、Razor 视图、插件设置页。
- `overwrite/`：整文件覆盖，最危险，逐个评估后重写。

结论：**文件夹插件（纯 htm + 静态）几乎零成本直搬；带逻辑的一律手工重写**。
本次迁移用同一套 SDK  covers 两种：文件夹插件免编译热插拔，代码插件走 `IXiunoPlugin` 接口。

## 2. SDK 形态（可直接照抄结构）

```text
src/XiunoBBS.Sdk/                 # 独立包：只依赖 ASP.NET Core 共享框架
  IXiunoPlugin.cs                 # Dir/Name/Brief/Version + ConfigureServices/MapEndpoints/
                                  # RenderSlot(point, sp)/InstallAsync/UninstallAsync/SettingUrl（后四个有默认实现）
  CodePluginHost.cs               # 反射发现 + 启动时显式装入 Xiuno.Plugin.*.dll（程序集懒加载，不预装就扫不到）
src/XiunoBBS.Core/Plugins.cs      # 文件夹插件：读 plugins/<dir>/plugin.json（含 enabled 开关）
                                  # hook/<point>.htm 片段拼接 + wwwroot/** 同步到站点
src/XiunoBBS.Services/
  PluginStateService.cs           # 代码插件启停 + 插件设置存 bbs_kv（键名前缀 plugin_state_/业务键原样保留）
src/XiunoBBS.Web/
  Helpers/PluginHtml.cs           # @Html.PluginSlot(point)：文件夹片段 + 代码插件 RenderSlot，统一按启用状态过滤，原样输出
  plugins/<dir>/                  # 文件夹插件本体（plugin.json + hook/ + wwwroot/ + 可选 install.sql）
  Views/AdminPlugin/Index.cshtml  # 统一列表：两类插件同表展示，启用/禁用/安装表/设置入口
src/Plugins/Xiuno.Plugin.<Dir>/   # 代码插件工程（net10 + FrameworkReference），Web 引用后即被发现
```

映射表（老插件→新位置，逐项打勾迁移）：

| PHP 侧 | C# 侧 |
|---|---|
| `conf.json` | 文件夹插件 `plugin.json`（字段相同）/ 代码插件接口属性 |
| `hook/*.htm` | 原样放入 `hook/`，挂载点名不变，视图里加 `@Html.PluginSlot("point")` |
| `hook/*.php` | 按挂载点语义重写：模板点→片段/Razor，逻辑点→服务方法/控制器动作 |
| `install.php` 纯 SQL 部分 | `install.sql`，后台一键执行；业务部分→ `InstallAsync` |
| `route/*.php` | 控制器动作或 `MapEndpoints` 的 Minimal API |
| `setting.php + setting.htm` | 插件自带设置页（如 `/plugin/<dir>/setting`，kv 存取） |
| `model/*.php` | Services 方法（含连带计数一并搬，见数据篇） |
| `overwrite/` | 不提供等价机制，逐个重写（明确记为差异） |

## 3. 挂载点只加不改，命名沿用原站

新挂载点直接用原站 hook 名（如 `body_start`、`footer_end`、`index_main_start`、`thread_user_after`），老插件作者一看就懂。
视图里只加一行 `@Html.PluginSlot("point")`，不碰原有结构。后台 `plugin-route` 映射同样走“旧形态→新控制器”中间件，与前台 URL 兼容同一写法。

## 4. 已验证的三个范例（与本次仓库同目录结构）

- `plugins/my_hello/`：单 `hook/body_start.htm`（上游原文），证明“HTML 片段零改动复用 + 启停即时生效”。
- `plugins/xn_scroll_top/`：`hook/footer_end.htm` + `wwwroot/1.js`（上游 JS 原样复用）+ 静态同步到 `/plugins/<dir>/`，证明静态资源链路。
- `src/Plugins/Xiuno.Plugin.InsertCode/`：对标上游 `xn_insert_code` 的完整代码插件——7 个 kv 键名原样保留、7 个挂载点同名映射、自带设置页。证明“有逻辑插件同模式迁移”。

## 5. 完成态声明（本案例已达成）

Xiuno 论坛迁移 + 插件迁移已完成，可无缝替换原版：

- 老数据库直挂：17 张表名列名与安装 SQL 逐字一致（大小写敏感比对），字符集用超集，`AuthKey` 对齐后老 cookie 继续有效。
- 老 URL 全兼容：前台双模式 + 后台链接全部改写到同一控制器，全量 200。
- 前台 + 后台 + 插件全量移植：含版主管理、附件、后台各模块、插件 SDK 与 3 个范例插件（`my_hello`、`xn_scroll_top`、`xn_insert_code`）。
- 判定“可替换”的门槛：自检页全绿 + 端到端回归全过 + 旧库挂载验证通过，三者缺一不可（见验证篇）。

## 6. 可复用的插件核对表

- 文件夹插件：清单字段齐全？hook 点名与视图槽一致？静态路径 `/plugins/<dir>/` 200？禁用后输出消失？
- 代码插件：程序集被宿主发现（输出目录有 dll 且启动扫描到）？启停走 kv？设置页仅管理员可进？
- `install.sql` 在空库执行一次通过；重复点击安装不炸（`IF NOT EXISTS` 或幂等 `InstallAsync`）。
- 每个搬过来的 hook 点：原站触发时机 == 新站渲染/执行时机（列表里注明差异点）。
- `overwrite/` 类需求必须单独列为“已知差异”，不允许静默丢功能。
