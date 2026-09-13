---
name: stos-setup
description: Use when the user wants to set up, configure, or connect StorytellerOS. Also triggers when stos_* tools return authentication errors, the connector says "Connection has expired" or won't authenticate, or the tools aren't appearing in your toolkit at all.
---

# StorytellerOS Setup

## Choose the client first

If the user is in **ChatGPT or Codex**, use the ChatGPT flow below and stop
before the Claude-specific instructions. If they are in Claude, keep using
the existing Claude flow. If the client is unclear, ask which app they are
connecting. Installing skills alone does not authenticate either connection.

### ChatGPT / Codex connection

1. Reuse an existing active credential pair from
   https://storytelleros.com/dashboard/settings/api-keys, or create a pair
   there if needed. Leave any working Claude connection untouched. A separate
   pair is optional for independent revocation, not required to add ChatGPT.
2. In desktop/Codex plugin settings, add a custom MCP named **StorytellerOS**,
   choose **Streamable HTTP**, and enter
   `https://storytelleros.com/api/mcp/chatgpt`. Leave bearer-token, headers,
   and environment variables empty. Choose **Automatic** or **CIMD**, not DCR.
3. Click **Authenticate**. Sign in on storytelleros.com, select the existing
   credential pair on the consent page, review access, and **Connect ChatGPT**.
   Do not ask for passwords, client secrets, or API keys in chat. ChatGPT does
   not need the pair's secret pasted anywhere; it stores and refreshes OAuth.
4. Start a new chat with the STOS connection enabled. Discover available tools
   if the client supports tool search, then call `stos_pen_names_list` as a
   read-only check. Missing tools alone do not prove a missing connection.

For ChatGPT web, custom connections require Developer mode and a supporting
plan/workspace policy. Use the same MCP URL and OAuth/CIMD. Web skill installs
are separate from desktop/CLI installs. Current user instructions:
https://storytelleros.com/docs/chatgpt.

If skills are missing, add the marketplace
`https://github.com/chellehoniker/claude-code-storytelleros` and install
**StorytellerOS**, where the client supports it. The terminal alternative is
`codex plugin marketplace add https://github.com/chellehoniker/claude-code-storytelleros`
then `codex plugin add storytelleros@storytelleros`. Run these in a terminal,
not chat. Restart the desktop app and open a new chat.

On an authentication error, retry Authenticate and check the selected pair's
revocation status and the account's active subscription. Do not rotate or
revoke a working Claude pair to fix a ChatGPT setup error. Revoking a shared
pair disconnects both clients. STOS currently uses the same account access
for both; do not invent per-credential pen-name permission controls.

## Claude-specific setup

## When to Use
- User says "set up storytelleros", "connect my workspace", "configure STOS"
- A `stos_*` tool returns 401 / "unauthorized" / "Invalid client credentials"
- The connector card shows "Connection has expired" or "Connection issue"
- The `stos_*` tools aren't appearing at all in your tool list
- User asks how to get started with StorytellerOS in Claude (for what-to-do-first coaching AFTER the connector works, hand off to the `getting-started` skill)

## How setup works

Two parts, intentionally split:

1. **The connector** — added manually via Cowork's "Add custom connector" dialog. Lives at `https://storytelleros.com/api/mcp` and authenticates with a Cowork connector credential pair the user generates from their dashboard. Cowork mints a Bearer token for each MCP request.
2. **The plugin** — installed via the marketplace. Adds 28 skills (`stos-setup`, `getting-started`, `pen-names`, `quick-task-capture`, `chapter-drafting`, `story-bible`, `worldbuilding`, `worldbuilding-merge`, `character-merge`, `markers`, `mentions`, `time-tracking`, `finance`, `calendar`, `timeline`, `tasks`, `manuscript-revisions`, `sales-studio`, `marketing-studio`, `social-handoff`, `blog-post-and-newsletter`, `wordpress-connect`, `wordpress-posts`, `wordpress-pages`, `wordpress-products`, `wordpress-media`, `wordpress-seo`, `wordpress-llm-optimization`) and 7 slash commands (`/stos-task`, `/stos-write`, `/stos-time`, `/stos-search`, `/stos-bible`, `/stos-finance`, `/stos-revise`). No connector bundled.

We deliberately do NOT bundle the connector in the plugin manifest. When a plugin auto-installs a connector, Cowork locks the credential fields, which blocks the user from pasting their own credentials. Splitting setup in two keeps those fields editable.

## Setup Flow

### Step 1 — Try a tool

Call `stos_tasks_list` (or any `stos_*` tool). Three possible outcomes:

**(a) Tool returns data or an empty list** — connected and authorized. Skip to Step 4.

**(b) Tool returns 401 / "Invalid client credentials"** — connector exists but credentials are wrong, missing, or revoked. Go to Step 2.

**(c) `stos_tasks_list` isn't an available tool at all** — no connector exists. Go to Step 3.

### Step 2 — Update the connector's credentials

Tell the user:

> "The connector needs a fresh Cowork connector credential pair. Open https://storytelleros.com/dashboard/settings/api-keys, find the **Cowork Connector** card, click **Generate credential pair**, and copy both values. The secret is shown only once — copy carefully."

Then walk them through pasting:

> "In Cowork: **Settings → Connectors**, click the **StorytellerOS** connector, scroll to **Advanced settings**, paste the new client id and secret, and click Save. The connector should re-authenticate within seconds."

If they still hit 401 after fresh credentials: check whether the credentials they're using were generated under a different account than the one currently signed into Cowork. Each credential pair is bound to a specific dashboard user.

### Step 3 — Add the connector for the first time

Tell the user:

> "First, generate a credential pair: open https://storytelleros.com/dashboard/settings/api-keys, find the **Cowork Connector** card, click **Generate credential pair**. Copy the client id (starts with `stcw_`) and secret (starts with `stcs_`). The secret is shown only once.
>
> Then in Cowork: **Settings → Connectors → Add custom connector**.
> - Name: **StorytellerOS**
> - Remote MCP server URL: `https://storytelleros.com/api/mcp`
> - Open **Advanced settings** and paste the client id and secret.
> - Click **Add**.
>
> The connector goes live within seconds. After it's added, run any StorytellerOS command and the tools should respond."

### Step 4 — Confirm

> "You're all set. Try `/stos-task hello from claude` to mint a quick task, or just ask me to 'list my books', 'start a writing timer for [project]', or 'log an expense for [vendor]'."

## Pairing with Author Automations Social

If the user has the Author Automations Social connector and plugin installed too, both work in the same Cowork conversation. StorytellerOS handles writing, knowledge, finance, calendar, and sales. AA Social handles all social posts, campaigns, and scheduling. After drafting a chapter or releasing a book, it's reasonable to suggest the user schedule promo via `aa_create_post` — StorytellerOS itself never wraps social.

## Troubleshooting Reference

### "Connection has expired" / "Connection issue"
The connector's tokens expired and refresh failed (usually because credentials were revoked or rotated). Walk through Step 2 — generate a fresh credential pair and paste it into the existing connector's Advanced settings.

### "Add custom connector" dialog has locked credential fields
This happens when a plugin auto-installs a connector — Cowork locks the fields because it thinks the plugin manages them. The StorytellerOS plugin does NOT bundle a connector specifically to avoid this. If the user sees locked fields, they're probably looking at a different plugin's connector entry. They should add a fresh one via the global "Add custom connector" path.

### "Tools worked yesterday, not today"
Most likely the user revoked the credential pair from the dashboard. Walk them through Step 2.

### "I have multiple pen names but only see data from one"
Profile-scoped tools default to the user's active / primary pen name. See the `pen-names` skill for the full multi-pen-name flow.

### "Update button is greyed out in Cowork"
Cowork's marketplace cache hasn't caught up. Three-dot menu on the marketplace (not the plugin) → **Check for updates**. If it stays greyed out for more than a day, remove and re-add the marketplace.

## What the user does NOT need to do

- Generate or paste a direct API key. That's for headless / scripted callers, not the Cowork connector.
- Run a browser approval flow. The connector uses the pasted credential pair directly.
- Edit any local config file. The plugin doesn't use one.
- Install Node.js, bun, or any binary.
