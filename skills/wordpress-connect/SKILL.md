---
name: wordpress-connect
description: Use when the user wants to connect their self-hosted WordPress site to StorytellerOS, troubleshoot a connection that isn't working, or check what's installed on the connected site. Triggers on "connect my WordPress", "add my WP site", "WordPress isn't connecting", "is my site connected?", "what plugins do I have on WordPress?".
---

# WordPress connection

The user connects their self-hosted WordPress site to StorytellerOS once, then every WordPress feature in StorytellerOS (Marketing Studio Posts/Pages, Sales Studio shop products, Media Studio library) acts on that site.

## How the connection works

Two methods, user picks one in **Settings → WordPress**:

| Method | When to recommend |
|---|---|
| **Application Password** | The fast path. Built into WordPress, no plugin install. Lets the user manage posts, pages, categories, tags, and media. Suggest this first. |
| **StorytellerOS plugin** | Only needed for shop management (WooCommerce / FluentCart products, orders, customers, coupons) and a few extras. The user installs the free `storytelleros.zip` on their site, generates a connection key, and pastes it into Settings → WordPress. |

Both methods are stored encrypted in the user's StorytellerOS account.

## Checking the connection

```js
stos_wp_site()
```

Returns site name, URL, WP version, plugin version (null if Application Password connection), and which commerce engines are installed (`hasWooCommerce`, `hasFluentCart`).

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
