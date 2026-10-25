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

## WordPress migration & compatibility

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
