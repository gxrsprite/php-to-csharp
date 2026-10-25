# 脱敏迁移经验：PHP Web 兼容核心

本记录来自两次 PHP 传统 Web 应用向 C#/.NET 迁移的案例（A类论坛型、B类 WordPress），只保留通用工程经验。B类部分已标注为 **WordPress 迁移经验**。已去除真实路径、真实密钥、真实账号与部署信息；以下向量均为现生成的示例。

## 1. 密码链：以后端实际收到的值为准（A类）

A类老站的登录/注册/安装共用一个公式：

- 前端 JS 先 `md5(明文)`（32 位小写 hex），POST 上来的是 hex；
- 后端 `password_db = md5(client_hex + salt)`，`salt` 为注册时生成的随机串；
- 安装脚本看似 `md5(md5(明文) + salt)`，与上式是同一个（因为它拿到的是明文）。

示例向量（`明文=test-pw-1`，`salt=Ab3k9XzqW2meP4rT`）：

- `md5(明文)` = `75ab2635679d1c6336a76ab064957435`
- `md5(hex+salt)` = `2738607fb681a254ef4c6de1d342fc4b`

C# 必须输出同样的值（UTF-8 编码、小写 hex）：

```csharp
using System.Security.Cryptography;
using System.Text;

public static class LegacyCrypto
{
    public static string Md5Hex(string s)
    {
        var bytes = MD5.HashData(Encoding.UTF8.GetBytes(s));
        var sb = new StringBuilder(32);
        foreach (var b in bytes) sb.Append(b.ToString("x2"));
        return sb.ToString();
    }

    public static string MakePassword(string clientMd5Hex, string salt) =>
        Md5Hex(clientMd5Hex.ToLower() + salt);

    public static bool VerifyPassword(string clientMd5Hex, string salt, string dbPassword) =>
        string.Equals(Md5Hex(clientMd5Hex.ToLower() + salt), dbPassword, StringComparison.OrdinalIgnoreCase);

    public static string TokenPwd(string dbPassword) => Md5Hex(dbPassword);
    public static string TokenKey(string authKey) => Md5Hex(authKey);
}
```

自检时用上面的示例向量断言 `MakePassword("75ab2635679d1c6336a76ab064957435", "Ab3k9XzqW2meP4rT")` 等于 `2738607fb681a254ef4c6de1d342fc4b`。

## 2. 对称加密 token：字节级移植（A类）

常见形态：明文 `"{ip}\t{time}\t{uid}\t{md5(password_db)}"`，经 XXTEA 加密 → base64 → 自定义 urlencode；key 为 `md5(auth_key)`。
PHP 的 `pack("V")` 是小端，整型是 32 位回绕语义，C# 用 `uint` + `unchecked` 等价实现：

```csharp
using System.Text;

public static class XxteaCompat
{
    private const uint Delta = 0x9E3779B9;

    private static uint[] Str2Long(byte[] s, bool includeLength)
    {
        int n = (s.Length + 3) / 4;
        var v = new uint[includeLength ? n + 1 : Math.Max(n, 1)];
        for (int i = 0; i < n; i++)
        {
            uint val = 0;
            for (int j = 0; j < 4; j++)
            {
                int idx = i * 4 + j;
                if (idx < s.Length) val |= (uint)s[idx] << (j * 8);
            }
            v[i] = val;
        }
        if (includeLength) v[n] = (uint)s.Length;
        return v;
    }

    private static byte[] Long2Str(uint[] v, bool includeLength)
    {
        int n = v.Length << 2;
        if (includeLength)
        {
            uint m = v[v.Length - 1];
            if (m > n) throw new InvalidOperationException("xxtea decrypt failed");
            n = (int)m;
        }
        var s = new byte[n];
        for (int i = 0; i < n; i++) s[i] = (byte)(v[i >> 2] >> ((i & 3) * 8));
        return s;
    }

    private static uint MX(uint z, uint y, uint sum, uint[] k, int p, uint e)
    {
        unchecked
        {
            return (((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4))) ^ ((sum ^ y) + (k[(p & 3) ^ e] ^ z));
        }
    }

    public static byte[] Encrypt(byte[] data, byte[] key)
    {
        if (data.Length == 0) return Array.Empty<byte>();
        var v = Str2Long(data, true);
        var k = Str2Long(key, false);
        Array.Resize(ref k, 4);
        unchecked
        {
            int n = v.Length - 1;
            uint z = v[n], y = v[0];
            uint q = (uint)Math.Floor(6 + 52.0 / (n + 1));
            uint sum = 0;
            while (q-- > 0)
            {
                sum += Delta;
                uint e = (sum >> 2) & 3;
                for (int p = 0; p < n; p++) { y = v[p + 1]; z = v[p] += MX(z, y, sum, k, p, e); }
                y = v[0]; z = v[n] += MX(z, y, sum, k, n, e);
            }
        }
        return Long2Str(v, false);
    }

    public static byte[] Decrypt(byte[] data, byte[] key)
    {
        if (data.Length == 0) return Array.Empty<byte>();
        var v = Str2Long(data, false);
        var k = Str2Long(key, false);
        Array.Resize(ref k, 4);
        unchecked
        {
            int n = v.Length - 1;
            uint z = v[n], y = v[0];
            uint q = (uint)Math.Floor(6 + 52.0 / (n + 1));
            uint sum = q * Delta;
            while (sum != 0)
            {
                uint e = (sum >> 2) & 3;
                for (int p = n; p > 0; p--) { z = v[p - 1]; y = v[p] -= MX(z, y, sum, k, p, e); }
                z = v[n]; y = v[0] -= MX(z, y, sum, k, 0, e);
                sum -= Delta;
            }
        }
        return Long2Str(v, true);
    }

    public static string EncryptText(string text, string key) =>
        Convert.ToBase64String(Encrypt(Encoding.UTF8.GetBytes(text), Encoding.UTF8.GetBytes(key)));

    public static string DecryptText(string base64, string key) =>
        Encoding.UTF8.GetString(Decrypt(Convert.FromBase64String(base64), Encoding.UTF8.GetBytes(key)));
}
```

外层的自定义 urlencode 也要逐行移植（base64 中的 `+/=` 会被改写，解码时先还原）：

```csharp
public static class LegacyUrlCodec
{
    public static string Encode(string s)
    {
        s = Uri.EscapeDataString(s);
        return s.Replace("_", "_5f").Replace("-", "_2d").Replace(".", "_2e")
            .Replace("+", "_2b").Replace("=", "_3d").Replace("%", "_");
    }
    public static string Decode(string s) => Uri.UnescapeDataString(s.Replace("_", "%"));
}
```

token 验签要复刻“超时回查”语义：token 内时间与现在差值超过阈值（如 30 分钟）时，回数据库比对 `md5(password_db)`，密码被修改则 token 失效。

## 3. 旧 URL 双模式：解析与生成（A类 `.htm` 形态）

老站通常有 rewrite 开/关两种形态，`?xxx-1.htm` 与 `xxx-1.htm` 都要能进同一控制器：

```csharp
public static class LegacyUrls
{
    public static string Url(string route, bool rewriteOn) =>
        rewriteOn ? $"{route}.htm" : $"?{route}.htm";

    public static string[] Parse(string? query, string? path)
    {
        string s = "";
        if (!string.IsNullOrEmpty(query) && query.Length > 1)
        {
            s = query[1..];
            int amp = s.IndexOf('&');
            if (amp >= 0) s = s[..amp];
        }
        else if (!string.IsNullOrEmpty(path) && path.EndsWith(".htm", StringComparison.OrdinalIgnoreCase))
            s = path.TrimStart('/')[..^4];
        if (s.EndsWith(".htm", StringComparison.OrdinalIgnoreCase)) s = s[..^4];
        return s.Length == 0 ? ["index"] : s.Split('-');
    }
}
```

映射到 MVC 时把分段同时放入具名参数（如 `tid`/`fid`，对应原 `param(1)/param(2)` 语义）与 `p0..pn` 保留位。
中间件必须注册在 `UseRouting` 之前，否则改写发生在选路之后，路径式 URL 会 404。

## 4. 登录态：session + token 双轨（A类）

入口逻辑是：`uid = session["uid"] ?? verify(token_cookie)`，命中 token 则回写 session。
C# 对应用 Session（cookie 名与 PHP 的 `sid` 名保持一致更利于过渡）+ 同名 token Cookie；退出时两边一起清。
`auth_key` 必须与旧站配置一致，否则存量 token 全部失效——这是上线前必查项。

## 5. B类站（WordPress 迁移经验：phpass/bcrypt 多轨密码 + HMAC cookie + 美化 permalink）

- 密码多轨验（对标 `wp_check_password`）：`长度很短 → md5(明文)裸哈希验`；`$P$/$H$ 头 → phpass 可移植哈希验`；`$wp 头 → base64(HMAC-SHA384(明文))再 BCrypt 验`；`$2y$ 头 → BCrypt 验`。新用户默认用 `$P$` 可移植哈希即可与旧环境互验（C# 侧用 BCrypt 兼容库验现代哈希）。
- phpass 字节级移植要点：`$P$B + 迭代指数 8 + 8 字节盐 + md5(salt+pw)迭代 2^8 次 + itoa64(./0-9A-Za-z)编码`，加密与编码函数逐行对齐；自检用“现算现验往返 + 上述示例 md5 向量”双断言锁死。
- 登录 cookie（对标 `wp_generate_auth_cookie`）：`login|exp|token|HMAC-SHA256(login|exp|token,key)`，`key=wp_hash(login|passfrag|exp|token,scheme)`；`passfrag` 规则是 `$P$/$2y$` 取 `substr(hash,8,4)`、否则取末 4 位；`wp_hash=HMAC-MD5(data, salt)`，盐按 `scheme`（auth/secure_auth/logged_in/nonce）取不同盐键。全套盐键必须与旧站 `wp-config.php` 一致。服务端 token 表存在用户元数据表（`session_tokens`），验签后要回查 token 子串（兼容旧序列化与新 JSON 两种存法）。
- 前端是**明文 POST**（与 A类前端预哈希不同，登录表单字段 `log/pwd`、注册 `user_login/user_email/user_pass` 原样提交），后端直接多轨验，不要加预哈希。
- URL 双模式（对标 `WP_Query` + `permalink_structure`）是朴素 `?p=/?page_id=/?s=/?cat=` 与美化 `/年/月/日/slug/`、`/{slug}/`、`/category/x/`、`feed`，生成走重写开关，解析恒双兼容。

## 6. 中间件放行表（WordPress 迁移经验）

只改写旧形态 URL 时，必须先放行 MVC 控制器首段与静态资源，否则 `/Compat` 会被当成内容 slug 改写而 404。
大小写敏感：小写 `feed/category/tag/search/author/page` 是旧站美化路径必须改写，大写首字母的才是 MVC 直达路径：

```csharp
private static readonly HashSet<string> MvcSegs = new(StringComparer.Ordinal)
{
    "Index", "Post", "Page", "Slug", "Search", "Archive", "Feed", "User", "Compat", "compat",
    "Media", "Admin", "Api",
    "css", "js", "lib", "images", "favicon.ico",
};
// + 登录/后台/接口等原版专用路径保持原路径
```

## 7. A类后台 token 与附件上传（论坛站迁移沉淀）

- 后台 token（`bbs_admin_token`）：明文 `"{ip}\t{time}"`，key=`md5(md5(useragent)+auth_key)`，对称加密后放 HttpOnly cookie；有效期按 3600 秒判，超 1800 秒滑动续发。**不要**复用前台 token 的 payload 格式，两者明文结构不同。
- 附件上传是 base64 形态：JS 先把文件读成 dataURL（图片还会在前端按宽高裁剪），POST 字段为 `name/data/width/height/is_image`（`data` 逗号后才是 base64 体）。服务端落盘到 `upload/tmp/`，会话里记 `tmp_files` 列表；发帖/回帖成功后才搬运到 `upload/attach/{年月}/` 并回填正文 URL、更新 `images/files` 计数。测试时用同一会话“先上传、再发帖”，断言附件行与 URL 回填。
- 中间件精确路径分支必须让位给旧形态解析：如 `/admin/` 精确匹配写在查询串判断之前，会把 `/admin/?index-login.htm` 吞掉造成登录页自环 302。精确分支要附加“无旧形态查询串”条件。

## 8. 可复用的兼容核对表

- 前端 POST 的密码是明文还是 md5 hex；后端公式里 salt 拼在前还是后。
- md5 输出大小写、输入编码（UTF-8）。
- token 明文字段顺序与分隔符、超时回查阈值、密码变更是否踢掉旧 token。
- 加解密底层是标准库还是自带实现（如 xxtea 纯 PHP 版），有无 `.so` 扩展分支。
- URL 生成与解析是否双模式；`param(n)` 的编号起点；ajax 与整页跳转是否走同一返回体。
- session cookie 名、token cookie 名、退出清理范围。
