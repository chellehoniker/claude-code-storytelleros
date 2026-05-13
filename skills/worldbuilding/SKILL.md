---
name: worldbuilding
description: Use when the user wants to add ONE character, location, lore entry, or story-timeline event — without spinning up a full story-bible flow. Triggers on "add a character to [title]", "describe this place", "add a new lore entry", "what's the origin of [thing]".
---

# Worldbuilding — single entries

## When to use this vs. `story-bible`

| Situation | Skill |
|---|---|
| Add one or two entries | `worldbuilding` |
| Build out a whole title's bible | `story-bible` |
| Refresh a single character's arc field | `worldbuilding` (then PATCH) |
| Extract everything from a manuscript | `story-bible` |

## Flow for one entry

1. Resolve scope: pen name + (optionally) title or series.
2. Draft the entry in the conversation — full fields, not a stub.
3. Show it to the user for approval.
4. Save: `stos_characters_create`, `stos_locations_create`, `stos_events_create`, or `stos_lore_create`.
5. If the entry has natural associations ("she's in chapter 3"), call `stos_worldbuilding_link` for each.

## Editing an existing entry

Use `_get` then `_update`:

```js
stos_characters_get({ id })
// review the current fields
stos_characters_update({ id, fields: { backstory: 'updated paragraph...' } })
```

## Anti-patterns

- **Creating a duplicate.** If the user says "add Sasha to *Curses and Currents*", check `stos_characters_list` first — Sasha may already exist under another title and just need a `stos_worldbuilding_link`.
- **Saving a stub.** A character record with only `character_name` is technically valid but useless. Draft the full record before saving.
