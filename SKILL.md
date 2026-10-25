---
name: php-to-csharp
description: "Migrate legacy PHP web apps (query-string routing, MySQL, cookie/session auth, embedded template views) to ASP.NET Core with FreeSql and MariaDB/MySQL, preserving URLs, password/token compatibility, data, and frontend appearance."
---

# PHP 传统 Web 应用转 C# 迁移

针对这一类：入口单文件 + 查询串路由、MySQL 直写 SQL、无框架模板视图（PHP/HTML 混写）、cookie + session 双轨登录。
先保外部可观察行为（URL、密码、token/cookie、数据、页面样式），再选 .NET 实现。
**先识别机制类型再选模板**：A类是前端预哈希（`md5(明文)` 再 POST，后端 `md5(hex+salt)`）、对称加密 token、`.htm` 双模式路由；
B类是 **WordPress 迁移经验**：明文 POST、多轨密码校验（裸哈希→可移植哈希→现代哈希逐级兼容）、HMAC cookie + 服务端 token 表、朴素 query + 美化路由双模式。不要把一套模板硬套到另一类站。

C类是 **Discuz! + UCenter**：明文 POST，UC 校验密码，论坛会员表另存 cookie 凭据；authcode + saltkey + 路径派生 cookie 前缀。涉及 Discuz! X3.5/X5 时先读 [Discuz 迁移经验](references/discuz-migration.md)，尤其注意 UID、帖子联合主键、CSS 分块、后台会话和 PHP 序列化设置。版块删除/合并需核对分表、关联清理及本地文件失败恢复。本分支验证首批论坛、后台详细版块设置与复制/合并/删除，仍不能据此认定整站已迁移。

## 工作流程

先固定参考源码目录并只读保留；确认 .NET SDK、FreeSql.Provider 与数据库驱动版本可还原。

1. **盘点行为契约**：读表结构描述文件（建表 + 种子）、登录注册流程（密码变换链：前端是否预哈希、后端是单公式还是多轨兼容）、token/cookie 生成验签（含密码片段规则/盐键/scheme）、路由分发（原单文件入口与查询类）、模板变量。记录每个兼容点，不猜测。
2. **建四层骨架**：Core（实体 + 密码/hash/加解密/URL 纯函数）/ Data（FreeSql 建表 + 种子）/ Services（按原数据层函数逐函数移植连带计数）/ Web（中间件 + 控制器 + Razor）。
3. **先过兼容自检**：用 `/Compat` 式自检页固定密码向量、加解密/cookie 往返、DB 版本、种子行；全绿再写业务。
4. **双模式路由兼容**：重写与查询串两种旧 URL 都要能进同一控制器；中间件必须放在 `UseRouting` 之前，**且必须放行 MVC 控制器首段与静态资源**（否则 `/Compat` 会被当成内容 slug 改写而 404，WordPress 迁移经验）。
5. **前端 1:1 还原**：静态资源原样搬运、模板逐页转 Razor（只换数据标签不动结构样式）、表单字段名与 JS 行为保持一致（A类是前端预哈希，B类是明文 POST，两者不要混）。
6. **端到端收尾**：老账号登录 + 注册 + 发帖 + 回帖 + 计数回查；区分“已验证等价”“已知差异”“未覆盖”。

字节兼容算法除自身往返外，还应以固定时钟/随机输入调用原 PHP 函数生成向量；账号与 cookie 流程尽量做 PHP→C#、C#→PHP 双向验证。自检全绿只说明已覆盖断言通过，不代表未迁移业务兼容。

## 核心提醒

- 密码链以后端实际收到的值为准：A类多是前端先哈希（`md5(明文)`）再 POST，后端只做 `md5(hex+salt)`；安装脚本看似双哈希，与上式是同一个（它拿到的是明文）。B类（WordPress 迁移经验）是明文 POST，后端走多轨兼容验（`wp_check_password`：裸哈希→phpass 可移植哈希→bcrypt 现代哈希）。先判定类型再实现，不要做成两种，也不要把明文站套成前端预哈希。
- token/cookie 必须字节级移植：对称加密注意大小端与整型溢出；HMAC 型注意密码片段规则（可移植哈希取固定偏移 4 位，否则取末 4 位）与 `scheme` 盐键。用自检向量锁死，不要“看起来像就行”。
- `auth_key`/盐密钥必须与旧站配置一致，否则老用户 cookie 全部失效。
- 新表字符集用旧库字符集的超集（如 `utf8` → `utf8mb4`）；存储引擎差异（MyISAM → InnoDB）只影响新建表，旧数据照读。
- 旧模板里的 `message(code, msg)` ajax 协议（`{code,message}`）是前端 JS 的契约，控制器要原样返回。
- 计数器（主题数/回帖数/今日数）是散在各 `*_create/*_delete` 里的连带更新，移植时逐个函数核对，不要只建主表。

涉及密码/token/URL/session 的字节级兼容，读 [PHP Web 兼容经验](references/php-web-compat.md)。

涉及建表、种子、计数器与旧库直挂，读 [数据层迁移经验](references/data-layer-migration.md)。

涉及模板、静态资源、表单 JS 与样式一致，读 [前端一致性经验](references/frontend-parity.md)。

涉及插件移植与通用插件机制，读 [插件化经验](references/plugin-system.md)。

涉及自检页、端到端脚本与收尾核对表，读 [验证经验](references/verification.md)。

涉及插件 SDK 与老插件迁移，读 [插件经验（Xiuno 迁移总结）](references/plugin-sdk.md)。

> 完成态（Xiuno 案例）：论坛前台 + 后台 + 插件 SDK 及 3 个范例插件已全部移植，
> 老数据库直挂、老 URL 全兼容，自检页全绿 + 端到端回归全过，可无缝替换原版。
