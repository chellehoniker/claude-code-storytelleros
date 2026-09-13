# StorytellerOS — Claude and ChatGPT Plugin

## ChatGPT / Codex setup

Version 0.7.0 adds ChatGPT setup using the same skills and hosted STOS tools.
Existing Claude installations keep their current connection and credentials.

Use the existing STOS account and credential pairs. Claude connections stay
unchanged; ChatGPT has its own OAuth endpoint on the same hosted MCP server.

1. Add a custom **Streamable HTTP** MCP server at
   `https://storytelleros.com/api/mcp/chatgpt`. Choose **Automatic/CIMD**, not
   DCR. Leave bearer-token and header fields empty.
2. Authenticate, sign in to StorytellerOS, select an existing credential pair,
   and approve. Do not paste its secret into chat. Revoking a shared pair
   disconnects both clients; a separate pair allows independent revocation.
3. Add this GitHub marketplace and install **StorytellerOS** for skills.
   With Codex CLI, run in your terminal:

```sh
codex plugin marketplace add https://github.com/chellehoniker/claude-code-storytelleros
codex plugin add storytelleros@storytelleros
```

Open a new chat with the connection enabled and ask to list pen names without
changing anything. The plugin does not auto-install the MCP connection.
Desktop/CLI skill installs do not automatically install skills on ChatGPT web.
Web custom connections depend on Developer mode and workspace policy.

For ChatGPT web: enable Developer mode in Settings → Security and login, open
[Plugins](https://chatgpt.com/plugins), select the plus button, and enter the
ChatGPT MCP URL under Connection. Authenticate through StorytellerOS, review
the discovered tools, and enable the connection in a new conversation. If your
workspace does not offer custom connections or marketplace imports, ask its
administrator; a desktop plugin installation does not enable web skills.

The 28 skills are shared with Claude. The seven `/stos-*` slash commands below
are Claude-specific shortcuts; describe the equivalent task in ordinary
language in ChatGPT. Available actions depend on current STOS permissions and
connected services, not merely on installing the plugin.

[Full ChatGPT instructions](https://storytelleros.com/docs/chatgpt).
This package is not a claim of public ChatGPT directory approval.

### Credentials and access

- Create or reuse a credential pair in STOS Settings → API Keys & Connectors.
  Select it during ChatGPT authorization; do not copy its secret into ChatGPT.
- Authentication is stored and refreshed, not repeated for every tool call.
  STOS still checks current account access and credential revocation.
- ChatGPT exposes the same STOS MCP catalog as Claude. This release does not
  introduce a separate pen-name permission system or a second hosted server.
- Keep Claude's URL at `/api/mcp`; use `/api/mcp/chatgpt` for ChatGPT.
- For independent disconnection, use separate credential pairs. Revoking a
  shared pair disconnects both clients.

### Troubleshooting

- **DCR not supported:** choose Automatic/CIMD, not dynamic registration.
- **Skills installed but no tools:** finish the separate MCP authentication,
  enable the connection, and start a new chat.
- **OAuth fails after consent:** confirm the server release and migration are
  complete, then check account access and whether the selected pair was revoked.
- **Enterprise domain restrictions unavailable:** this connection does not
  implement OIDC; workspaces requiring it may block installation.
- Share the error text with support, never passwords, secrets, or tokens.

Client screens can change. See OpenAI's official
[connection guide](https://developers.openai.com/plugins/deploy/connect-chatgpt)
and [authentication reference](https://developers.openai.com/plugins/build/auth).

## Claude setup (unchanged)

Drive your StorytellerOS workspace from inside Claude Code or Claude Cowork — drafting and revising chapters, generating full story bibles, looking up characters and lore, capturing tasks, running writing timers, logging expenses, and managing your calendar without leaving the conversation.

> StorytellerOS calls long-form works **titles**, not books — that covers novels, screenplays, audiobook scripts, games, and novellas equally.

## Install

The plugin ships **skills and slash commands**. The MCP connector itself is added separately so the credential fields stay editable. Two short steps:

### Step 1 — Add the connector

In Cowork (or Claude Desktop): **Settings → Connectors → Add custom connector**

- **Name:** `StorytellerOS`
- **Remote MCP server URL:** `https://storytelleros.com/api/mcp`
- **Advanced settings → client id:** generate at [storytelleros.com/dashboard/settings/api-keys](https://storytelleros.com/dashboard/settings/api-keys) → **Cowork Connector** card → **Generate credential pair** → copy the value starting with `stcw_`
- **Advanced settings → client secret:** the value starting with `stcs_` (shown once, copy carefully)
- Click **Add**.

The connector authenticates immediately and the `stos_*` tools become available.

### Step 2 — Install the plugin (for skills + slash commands)

In any Claude Code or Cowork session:

```
/plugin marketplace add https://github.com/chellehoniker/claude-code-storytelleros
/plugin install storytelleros@storytelleros
```

This adds the conversational skills (see the table below) and the `/stos-*` slash commands. No connector is bundled — Step 1 already covers that.

## Quick start

```
/stos-task remember to email the cover designer about the rebrand
```

Or just describe what you want:

> "Draft chapter 4 of my Indie Annie cozy — the previous chapter ended with the cat knocking the urn over."

> "Build a full story bible for *Curses and Currents* from the manuscript I just uploaded."

> "Start a writing timer for the Stardew prequel."

> "Log a $400 expense for editor Sarah Marsh, category Editing, against *Curses and Currents*."

> "Post about my new release on Instagram." → see the legacy social-skill caveat under *Social work* before testing.

Claude pulls the relevant story bible, writes in the right pen name's voice, and saves work back to your workspace. You review before anything is finalized.

## What's bundled

| Component | Triggers / Notes |
|---|---|
| **MCP tools** | Supplied by the separately authenticated hosted connection, not bundled in this package. Both clients use the current STOS tool catalog. |
| **Skills** | 28 shared skills for setup, writing, story bibles, tasks, time, finance, marketing, and WordPress. See the [skills directory](skills/) for the complete list. |
| **Slash commands** | `/stos-task <text>`, `/stos-write <description>`, `/stos-time <start\|stop\|status>`, `/stos-search <query>`, `/stos-bible <title>`, `/stos-finance <income\|expense> <amount> <note>`, `/stos-revise <chapter> <feedback>` |

## Story bibles — built one entry at a time

The `story-bible` skill walks Claude through generating a full bible (characters, locations, events, lore) for a title and saves **each entry as its own POST** — no batching, no truncation. For large manuscripts with 50+ characters, that's 50+ individual `stos_characters_create` calls, then 50+ `stos_worldbuilding_link` calls to wire them into scenes. Slower than a bulk upload but every field arrives intact.

## Social work — current tools and legacy skill caveat

The hosted STOS catalog now includes social and ads tools; see the
[current Social Studio contract](https://storytelleros.com/docs/claude-cowork/social-handoff).
However, the bundled legacy `social-handoff` skill still directs the assistant
to a separate Reader Radius connection using `aa_*` tools. Until that skill is
updated and tested, do not assume a social request will use the STOS connection.
Its existing flow:

1. Calls `stos_pen_names_get` to read the pen name's `aaProfileId` (set automatically by the AA ↔ STOS sync).
2. Stops with a clear message if the pen name isn't connected to AA Social.
3. Otherwise calls the matching `aa_*` tool with `profileId: aaProfileId`.

This is a legacy skill dependency, not a requirement for every STOS tool.
Review the selected tool, account, content, schedule, and any spend before
authorizing a social action.

## Claude authentication

Authentication is on the **connector** (added in Step 1 of Install), not the plugin. You paste a credential pair you generate from your StorytellerOS dashboard into the connector's Advanced settings.

Lost the secret? Generate a new pair from [Settings → API keys](https://storytelleros.com/dashboard/settings/api-keys), then update the connector in Cowork: Settings → Connectors → click the connector → paste the new credentials → Save. The old pair stays revocable from the same Settings card.

If you have multiple pen names, your connector exposes every pen name on your account. Switch between them by saying *"draft under my [pen name]"* or *"switch to [pen name]"* — the bundled `pen-names` skill teaches Claude how to discover your pen names and route subsequent calls. See `skills/pen-names/SKILL.md`.

## Optional Reader Radius connection

Reader Radius (formerly Author Automations Social) is a separate product at
[readerradius.com](https://readerradius.com). Its connection is authenticated
separately; installing StorytellerOS does not install or authorize it. See the
legacy social-skill caveat above before testing combined workflows.

## Troubleshooting

- **`stos_*` calls return 401:** run `/stos-setup` or just say "set up storytelleros" — the `stos-setup` skill walks through generating fresh credentials and updating the connector.
- **Tools not showing up at all:** the connector probably wasn't added (Step 1). Add it via Settings → Connectors → Add custom connector.
- **Wrong pen name's data appearing:** see the `pen-names` skill — pass the `penNameId` argument on every call in a multi-step flow.
- **Social handoff asks for another plugin:** the legacy skill expects separate Reader Radius tools. See *Social work* above and the current STOS social-tool guide; do not paste credentials into chat to work around it.

## Updating

### ChatGPT / Codex

Use the marketplace update controls in the client where you installed
StorytellerOS, then start a new conversation. Update the skills plugin and MCP
metadata separately: in ChatGPT web, open Plugins, select the MCP connection,
and choose Refresh. Reauthenticate if requested; do not generate a new pair
just to update skills. Keep Claude's existing connection unchanged.

See [OpenAI's connection and refresh guide](https://developers.openai.com/plugins/deploy/connect-chatgpt).

### Claude Cowork

Settings → Plugins → three-dot menu next to the marketplace → toggle **Sync automatically** ON. To force a check, click **Check for updates**.

### Claude Code (CLI)

```
/plugin update storytelleros@storytelleros
```

## Requirements

- An active [StorytellerOS](https://storytelleros.com) account
- The authenticated connector for your client, installed separately from skills
- A client/workspace that permits custom MCP connections and, for skills,
  marketplace installation. Public ChatGPT directory approval is not implied.

## Support

- Email: support@storytelleros.com
- Bugs / feature requests: [open an issue](https://github.com/chellehoniker/claude-code-storytelleros/issues)

## License

MIT
