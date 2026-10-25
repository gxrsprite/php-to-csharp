# php-to-csharp

[中文文档](README.zh-CN.md)

Migrate legacy PHP web apps to C# (ASP.NET Core + FreeSql + MariaDB/MySQL),
preserving externally observable behavior — URLs, passwords/tokens, data, frontend —
then choose the .NET implementation.

## Project types (classify first, never mix templates)

- **Type A**: single-entry + query-string routing, raw MySQL, framework-less views,
  cookie + session dual login, client-side pre-hash (`md5(plaintext)` POSTed),
  symmetric-encrypted tokens.
- **Type B (WordPress migration experience)**: plain-text POST, multi-track password
  verification (raw → portable → modern hashes), HMAC cookie + server-side token table,
  plain-query + pretty-permalink dual routing.

## Part 1: WordPress migration (Type B)

A full WordPress → C# migration was completed with this skill. Compatibility points:

- **Schema**: all 12 tables modeled 1:1
  (`wp_posts/postmeta/users/usermeta/terms/term_taxonomy/relationships/termmeta/
  comments/commentmeta/options/links`) — names, fields, keys match the originals;
  legacy databases mount read-only; charset `utf8mb4`.
- **Passwords**: three-track verification mirroring `wp_check_password`
  (raw hash → phpass portable hash → bcrypt modern hash, incl. the `$wp` SHA-384 pre-hash);
  new users default to the portable hash; verified both directions against PHP.
- **Sessions**: `login|exp|token|HMAC-SHA256` cookies mirroring
  `wp_generate_auth_cookie` (incl. passfrag and per-scheme salt rules) plus
  server-side token-table lookup. All eight salt keys must match `wp-config.php`,
  otherwise existing cookies die.
- **Routing**: dual plain (`?p=/?page_id=/?s=/?cat=`) and pretty
  (`/yyyy/mm/dd/slug/`, category/tag/date/author/attachment) parsing; one generator
  driven by the rewrite switch. Middleware must allow-list MVC segments
  (case-sensitive) and static assets, and run before `UseRouting`.
- **Content**: `the_content` fallback chain (meta → excerpt → template → truncation),
  shortcode stripping, wpautop paragraphing, comment `kses` filtering.
  Razor's default encoder turns CJK into `&#x;` entities while PHP emits raw UTF-8 —
  relax the encoder to the full Unicode range.
- **Feeds & crawling**: RSS2/Atom/comment feeds, sitemap (split posts/pages), robots.txt
  (incl. custom rules).
- **API & admin**: read-only `/wp-json/wp/v2` (posts/pages/media/comments/users/
  categories/tags/search/types/statuses) + comment creation; minimal admin loop
  (dashboard/posts/pages/comment moderation/users/settings/uploads); legacy paths kept
  (`wp-login.php`, `wp-comments-post.php`, `wp-admin`, standard xmlrpc fault).
- **Plugins**: Rank Math SEO ported by "frontend contract + storage keys"
  (full head tags, JSON-LD graph, breadcrumbs, 301/302/307/410/451 redirections,
  admin SEO pages) — see [plugin-system](references/plugin-system.md) and
  [frontend-parity §6](references/frontend-parity.md).

- **Plugins**: Rank Math SEO ported by "frontend contract + storage keys"
  (full head tags, JSON-LD graph, breadcrumbs, 301/302/307/410/451 redirections,
  admin SEO pages) — see [plugin-system](references/plugin-system.md) and
  [frontend-parity §6](references/frontend-parity.md).

---

## Part 2: Xiuno forum migration (Type A)

A full Xiuno BBS 4.0 → C# migration was completed with this skill:

- **Passwords**: frontend pre-hashes with `md5(plaintext)` before POST; backend only does
  `md5(hex+salt)` (the install script's double hash is the same formula — it receives plaintext).
  Default admin `admin / 1` logs in against the legacy DB as-is.
- **Sessions**: session uid + `bbs_token` dual track; token plaintext
  `"$ip\t$time\t$uid\tmd5(password)"` via XXTEA (little-endian + int32 semantics) +
  base64 + custom urlencode, key `md5(auth_key)`, password re-check past 30 min;
  separate `bbs_admin_token` for admin (`"$ip\t$time"`, key `md5(md5(useragent)+auth_key)`,
  3600 s expiry with 1800 s sliding refresh).
- **Routing**: dual `?thread-1.htm` / `thread-create-1.htm` plus admin `?forum-list.htm`,
  all rewritten to the same controllers; middleware runs before `UseRouting`;
  exact-path branches must defer to legacy-shape parsing
  (`/admin/` swallowing the query string caused a login-page self-redirect loop).
- **Data**: all 17 tables/columns match install SQL verbatim (case-sensitive comparison;
  ORM PascalCase defaults are the #1 trap); charset superset `utf8mb4`;
  `thread_create/post_delete/user_delete` cascading counts ported per function.
- **Frontend**: assets copied verbatim, templates ported to Razor (data tags only),
  form field names and JS untouched, ajax `{code,message}` protocol preserved;
  form JS must be emitted via sections (inline scripts run before jQuery and die silently,
  leaving a native submit that prints raw JSON); quick-reply `return_html` renders
  the `<li>` server-side so the original JS stays byte-identical.
- **Coverage**: 13 frontend page types, mod top/move/delete/close, attach upload/delete/download,
  9 admin modules (login/home/forums/groups/users/thread search/settings/SMTP/cache/plugins).

### How plugins stay compatible (the key part of this case)

PHP plugins are just three things, each with a fixed migration path — no automatic conversion exists:

| PHP side | C# side | Cost |
|---|---|---|
| `conf.json` manifest | Folder-plugin `plugin.json` (same fields) / code-plugin interface props | Copy |
| `hook/*.htm` pure HTML | Verbatim into `hook/`, same slot names, one `@Html.PluginSlot("point")` per view | **Zero — reuse as-is** (`my_hello` example) |
| Static files | Synced at startup to `wwwroot/plugins/<dir>/`, served at `/plugins/<dir>/...` | Copy (`xn_scroll_top` example, JS reused verbatim) |
| `hook/*.php` | Rewritten per slot semantics: template slots → fragments/Razor, logic slots → service methods/controller actions | Manual, one by one |
| Pure-SQL `install.php` | `install.sql`, one-click execution from admin; logic parts → `InstallAsync` | Low |
| `route/*.php` | Controller actions or Minimal API `MapEndpoints` | Medium |
| `setting.php + setting.htm` | Plugin-owned settings page (e.g. `/plugin/<dir>/setting`, kv storage, keys kept verbatim) | Medium (`xn_insert_code` example: 7 keys, 7 slots, isomorphic port) |
| `model/*.php` | Service methods (cascading counts included) | Medium |
| `overwrite/` full-file replacement | **No equivalent** — rewritten case by case, recorded as known diffs | High |

Supporting kit (`src/XiunoBBS.Sdk` package + unified admin page):

- `IXiunoPlugin`: `Dir/Name/Brief/Version` + `ConfigureServices / MapEndpoints /`
  `RenderSlot / InstallAsync / UninstallAsync / SettingUrl` (latter four have defaults).
- `CodePluginHost`: reflection discovery + explicit preload of `Xiuno.Plugin.*.dll`
  (assemblies are lazy-loaded — undiscovered if not preloaded).
- `PluginStateService`: code-plugin enable/disable + settings in `bbs_kv`.
- Admin plugins page: unified folder/code list with enable/disable/install/settings.
- Slot names reuse the original hook names (`body_start`, `footer_end`,
  `index_main_start`, `thread_user_after`, ...).
- Same trust model as PHP: only admins can install/enable plugins; fragments render raw.

---

## Done criteria (Xiuno + WordPress cases)

Frontend + admin + plugins fully ported; legacy DB mounts, legacy URLs all work;
compat page green + end-to-end regression green — drop-in replaceable.
Replaceable = green compat + green regression + mounted-legacy-DB verification.
All three are required.

## Quick use

```text
Use $php-to-csharp to plan and implement this PHP-to-C# migration,
preserving existing behavior contracts first.
```

Or load `php-to-csharp` via the skill tool (full instructions in [SKILL.md](SKILL.md));
IDE default prompt in [agents/openai.yaml](agents/openai.yaml).

## Workflow (summary; details in SKILL.md)

1. Inventory behavior contracts (schema, password chain, tokens/cookies, routing, template vars) — no guessing.
2. Four-layer skeleton: Core (pure functions) / Data (tables + seeds) / Services (per-function port) / Web (middleware + controllers + Razor).
3. Pass the compat self-check page (`/Compat`-style) first; only then build features.
4. Dual-mode routing; middleware before `UseRouting`, allow-listing MVC segments and static assets.
5. Pixel/behavior frontend parity (assets copied verbatim, templates swap data tags only, form JS untouched); port plugins by frontend contract + storage keys.
6. Finish end-to-end; report verified-equivalent / known-diffs / uncovered.

## Docs

| Doc | Covers |
|---|---|
| [SKILL.md](SKILL.md) | Full instructions, workflow, reminders, branch index |
| [references/php-web-compat.md](references/php-web-compat.md) | Byte-level password/token/URL/session compat, Type A + WordPress templates |
| [references/data-layer-migration.md](references/data-layer-migration.md) | Tables, seeds, cascading counts, legacy-DB mounting |
| [references/frontend-parity.md](references/frontend-parity.md) | Templates, assets, form JS, Razor pitfalls, plugin ports |
| [references/plugin-system.md](references/plugin-system.md) | Plugin system: hook bus, modules, UI zones |
| [references/plugin-sdk.md](references/plugin-sdk.md) | Plugin SDK migration incl. PHP→C# mapping |
| [references/verification.md](references/verification.md) | Compat page, e2e scripts, sign-off checklist |
| [agents/openai.yaml](agents/openai.yaml) | IDE default prompt |
