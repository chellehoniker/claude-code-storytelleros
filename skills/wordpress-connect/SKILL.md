---
name: wordpress-connect
description: Use when the user wants to connect their self-hosted WordPress site to StorytellerOS, troubleshoot a connection that isn't working, or check what's installed on the connected site. Triggers on "connect my WordPress", "add my WP site", "WordPress isn't connecting", "is my site connected?", "what plugins do I have on WordPress?".
---

# WordPress connection

The user connects their self-hosted WordPress site to StorytellerOS once, then every WordPress feature in StorytellerOS (Marketing Studio Posts/Pages, Sales Studio shop products, Media Studio library) acts on that site.

## How the connection works

One method: the free **StorytellerOS plugin** on the user's site. The user installs `storytelleros.zip` (downloadable from Settings → WordPress), opens **StorytellerOS** in their WP sidebar, clicks **Generate token**, and pastes the `stos_` connection key into StorytellerOS Settings → WordPress. That powers everything — posts, pages, media, categories/tags, and shop management (WooCommerce / FluentCart).

Older connections may still use a WordPress Application Password. They keep working, but the app no longer offers that method — when one plays up, the fix is switching it to a connection key (Settings → WordPress → Edit on that connection).

## Multiple sites (one per pen name)

Users can connect one WordPress site per pen name, plus an **account default** used by pen names without their own site. Every `stos_wp_*` tool takes an optional `penNameId` (get ids from `stos_pen_names_list`); omitted = account default. A pen name without its own site automatically uses the account default. If `stos_wp_site({ penNameId })` 404s with a pen-name message, that pen name id doesn't exist on the account.

## Checking the connection

```js
stos_wp_site({ penNameId? })
```

Returns site name, URL, WP version, plugin version, and which commerce engines are installed (`hasWooCommerce`, `hasFluentCart`). Plugin version is only null for a legacy Application Password connection that hasn't been switched over yet.

If the call returns 404 with "no WordPress site is connected", walk the user through Settings → WordPress.

## Common failure modes

When `stos_wp_site` or any other `stos_wp_*` tool fails, the error message names the security plugin when it can detect one:

- **Wordfence** — usually OK out of the box. If "Disable application passwords" is on, ask the user to flip it off.
- **iThemes / Solid Security** — if "Restrict REST API" is on, set it to Default Access.
- **All-In-One WP Security** — disable "Disallow access to WordPress REST API" under Firewall → Basic Firewall.
- **WP Cerber** — disable "Disable REST API" under Hardening.
- **Cloudflare or other WAF** — the user may need a rule that lets StorytellerOS through.

The error message itself usually has the right wp-admin path. Surface it to the user verbatim instead of paraphrasing.

## What this skill does NOT do

- Modifying the user's WordPress site password — never. Application Passwords are the right tool.
- Reading or storing the user's main WordPress login credentials.
- Any direct database access to WordPress.

## Anti-patterns

- Do not generate or fabricate a plugin token. The user gets one from wp-admin → StorytellerOS → Generate token, and it's shown only once.
- Do not paste a token into a chat reply if the user shares one — confirm and remind them to keep it private.
