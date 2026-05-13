# StorytellerOS — Claude Plugin

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

> "Post about my new release on Instagram." → routes to the Author Automations plugin (see *Social work* below).

Claude pulls the relevant story bible, writes in the right pen name's voice, and saves work back to your workspace. You review before anything is finalized.

## What's bundled

| Component | Triggers / Notes |
|---|---|
| **MCP tools** | ~80 `stos_*` tools mirroring `/api/v1/*` — full CRUD across titles, chapters, scenes, characters, locations, lore, events, tasks, calendars, time-tracking, finance, sales, marketing, settings. |
| **Skills** | `stos-setup`, `pen-names`, `quick-task-capture`, `chapter-drafting`, `story-bible`, `worldbuilding`, `time-tracking`, `finance`, `calendar`, `timeline`, `tasks`, `manuscript-revisions`, `sales-studio`, `marketing-studio`, `social-handoff` |
| **Slash commands** | `/stos-task <text>`, `/stos-write <description>`, `/stos-time <start\|stop\|status>`, `/stos-search <query>`, `/stos-bible <title>`, `/stos-finance <income\|expense> <amount> <note>`, `/stos-revise <chapter> <feedback>` |

## Story bibles — built one entry at a time

The `story-bible` skill walks Claude through generating a full bible (characters, locations, events, lore) for a title and saves **each entry as its own POST** — no batching, no truncation. For large manuscripts with 50+ characters, that's 50+ individual `stos_characters_create` calls, then 50+ `stos_worldbuilding_link` calls to wire them into scenes. Slower than a bulk upload but every field arrives intact.

## Social work — handed off to Author Automations

This plugin does **not** post to social. Social is handled by the [Author Automations Social](https://authorautomations.social) plugin, which already has 22 `aa_*` tools across 15 platforms. When you ask Claude to post, schedule, or run a campaign, the `social-handoff` skill:

1. Calls `stos_pen_names_get` to read the pen name's `aaProfileId` (set automatically by the AA ↔ STOS sync).
2. Stops with a clear message if the pen name isn't connected to AA Social.
3. Otherwise calls the matching `aa_*` tool with `profileId: aaProfileId`.

No double-hop, no STOS-side social proxy — the two plugins work side by side in the same conversation.

## Authentication

Authentication is on the **connector** (added in Step 1 of Install), not the plugin. You paste a credential pair you generate from your StorytellerOS dashboard into the connector's Advanced settings.

Lost the secret? Generate a new pair from [Settings → API keys](https://storytelleros.com/dashboard/settings/api-keys), then update the connector in Cowork: Settings → Connectors → click the connector → paste the new credentials → Save. The old pair stays revocable from the same Settings card.

If you have multiple pen names, your connector exposes every pen name on your account. Switch between them by saying *"draft under my [pen name]"* or *"switch to [pen name]"* — the bundled `pen-names` skill teaches Claude how to discover your pen names and route subsequent calls. See `skills/pen-names/SKILL.md`.

## Pairs with Author Automations Social

Install both plugins side by side. StorytellerOS handles writing, knowledge, finance, calendar, and sales. Author Automations handles posts, campaigns, and scheduling. Both work in the same Cowork conversation — Claude routes social work to `aa_*` tools and writing/business work to `stos_*` tools.

## Troubleshooting

- **`stos_*` calls return 401:** run `/stos-setup` or just say "set up storytelleros" — the `stos-setup` skill walks through generating fresh credentials and updating the connector.
- **Tools not showing up at all:** the connector probably wasn't added (Step 1). Add it via Settings → Connectors → Add custom connector.
- **Wrong pen name's data appearing:** see the `pen-names` skill — pass the `penNameId` argument on every call in a multi-step flow.
- **Social handoff stops because pen name isn't connected:** open Author Automations and provision the pen name there; STOS picks up the link via webhook automatically.

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
