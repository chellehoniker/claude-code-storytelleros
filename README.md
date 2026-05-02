# StorytellerOS — Claude Plugin

Drive your StorytellerOS workspace from inside Claude Code or Claude Cowork — drafting and revising chapters, looking up characters and lore, capturing tasks, running writing timers, logging expenses, and managing your calendar without leaving the conversation.

> **Pre-1.0.** This plugin ships skills and slash commands that talk to the StorytellerOS hosted MCP endpoint. The endpoint is rolling out alongside this plugin; if your `stos_*` tool calls return 401 right after install, the connector and endpoint are still being wired up — try again shortly or reach out to support.

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

This adds the conversational skills (`stos-setup`, `pen-names`, `quick-task-capture`) and the `/stos-task`, `/stos-write`, `/stos-time`, `/stos-search` slash commands.

## Quick start

```
/stos-task remember to email the cover designer about the rebrand
```

Or just describe what you want:

> "Draft chapter 4 of my Indie Annie cozy — the previous chapter ended with the cat knocking the urn over."

> "Start a writing timer for the Stardew prequel."

> "Search my lore for everything about the Crow Court."

> "Log a $400 expense for editor Sarah Marsh, category Editing."

Claude pulls the relevant story bible, writes in the right pen name's voice, and saves work back to your workspace. You review before anything is finalized.

## What's bundled

| Component | Notes |
|---|---|
| **Skills** | `stos-setup` (connect & troubleshoot), `pen-names` (switch between author identities), `quick-task-capture` (one-shot todos) |
| **Slash commands** | `/stos-task <text>`, `/stos-write <description>`, `/stos-time <start\|stop\|status>`, `/stos-search <query>` |

The fuller skill bundle — chapter drafting, world-building, time-tracking review, finance flows — lands in a future release as the corresponding tool surface goes live.

## Pairs with Author Automations Social

If you also use [Author Automations Social](https://authorautomations.social) for social posts, install both plugins side by side. StorytellerOS handles writing, knowledge, finance, calendar, and sales. Author Automations handles posts, campaigns, and scheduling. Both work in the same Cowork conversation — Claude will route social work to `aa_*` tools and writing/business work to `stos_*` tools.

## Authentication

Authentication is on the **connector** (added in Step 1 of Install), not the plugin. You paste a credential pair you generate from your StorytellerOS dashboard into the connector's Advanced settings.

Lost the secret? Generate a new pair from [Settings → API keys](https://storytelleros.com/dashboard/settings/api-keys), then update the connector in Cowork: Settings → Connectors → click the connector → paste the new credentials → Save. The old pair stays revocable from the same Settings card.

If you have multiple pen names, your connector exposes every pen name on your account. Switch between them by saying *"draft under my [pen name]"* or *"switch to [pen name]"* — the bundled `pen-names` skill teaches Claude how to discover your pen names and route subsequent calls. See `skills/pen-names/SKILL.md`.

## Troubleshooting

- **`stos_*` calls return 401:** run `/stos-setup` or just say "set up storytelleros" — the `stos-setup` skill walks through generating fresh credentials and updating the connector.
- **Tools not showing up at all:** the connector probably wasn't added (Step 1). Add it via Settings → Connectors → Add custom connector.
- **Wrong pen name's data appearing:** see the `pen-names` skill — pass the pen-name selector on every call in a multi-step flow.

## Updating

### Claude Cowork

Settings → Plugins → three-dot menu next to the marketplace → toggle **Sync automatically** ON. To force a check, click **Check for updates**.

### Claude Code (CLI)

```
/plugin update storytelleros@storytelleros
```

## Requirements

- An active [StorytellerOS](https://storytelleros.com) account
- The Cowork connector added per Step 1 above

## Support

- Email: support@storytelleros.com
- Bugs / feature requests: [open an issue](https://github.com/chellehoniker/claude-code-storytelleros/issues)

## License

MIT
