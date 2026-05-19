---
name: worldbuilding
description: Use when the user wants to ADD or EDIT ONE character, location, lore entry, or story-timeline event — without spinning up a full story-bible flow. Triggers on "add a character to [title]", "save this location to the bible", "create a new lore entry for [thing]", "update [character]'s backstory", "describe a place and save it". For merging or deduplicating, use the `character-merge` skill instead.
---

# Worldbuilding — single entries

## When to use this vs. other skills

| Situation | Skill |
|---|---|
| Add one or two entries | `worldbuilding` |
| Edit one field on an existing record | `worldbuilding` |
| Build out a whole title's bible | `story-bible` |
| Extract everything from a manuscript | `story-bible` |
| Merge / dedupe / consolidate characters | **`character-merge`** (never per-record `_get` in a loop) |

## Canonical field values — send these exactly

The Story Bible tables have strict CHECK constraints. Sending anything other than the canonical value below silently drops the field — the rest of the record saves, but you'll have wasted tokens and the user will see a blank column in the UI. **Map synonyms to the canonical value before calling the tool. If a value can't be mapped, omit the field entirely — never invent, never stash the original somewhere else.**

| Field | Canonical values | Synonym → canonical |
|---|---|---|
| `characters.role_in_story` | `protagonist`, `antagonist`, `supporting`, `minor`, `love-interest`, `other` | hero / heroine / lead / main / MC → **protagonist** • villain / big-bad / rival → **antagonist** • deuteragonist / mentor / sidekick / ally / foil / friend / family → **supporting** • love interest / romantic interest / romance → **love-interest** • background / extra / NPC / cameo / walk-on → **minor** |
| `characters.life_status` | `alive`, `dead`, `undead`, `missing`, `unknown` | deceased / killed → **dead** • living / alive-and-well → **alive** • absent / lost / disappeared → **missing** • ghost / vampire / zombie → **undead** |
| `characters.pov_status` | `pov-character`, `non-pov` | any POV / occasional-POV / multi-POV / "yes" / "true" / "POV" → **pov-character** • everything else (or "no" / "false") → **non-pov** |
| `events.act` | `act-one`, `act-two`, `act-three`, `act-four` | 1 / I / "Act 1" / "Act I" → **act-one** (2/II → act-two, 3/III → act-three, 4/IV → act-four). Map non-structural beats (climax / midpoint / dark night / inciting incident) to the closest structural act and put the original beat label in `description`. |

**Never send `custom_fields`** on a Cowork worldbuilding create call. The StorytellerOS UI does not render those keys for Cowork-completed records — the data lands invisible. Stick to the named columns listed in each tool's writable-fields description.

## Flow for one entry

1. Resolve scope: pen name + (optionally) title or series.
2. Draft the entry in the conversation — full fields, not a stub.
3. Show it to the user for approval.
4. Save: `stos_characters_create`, `stos_locations_create`, `stos_events_create`, or `stos_lore_create`.
5. If the entry has natural associations ("she's in chapter 3"), call `stos_worldbuilding_link` for each.

## Editing an existing entry

If you need ONE record, use `_get` then `_update`:

```js
stos_characters_get({ id })
// review the current fields
stos_characters_update({ id, fields: { backstory: 'updated paragraph...' } })
```

**If you need TWO OR MORE records**, do NOT loop `_get`. For characters specifically, use `stos_characters_get_many({ ids: [...] })` — one round-trip instead of N. This matters because every MCP round-trip replays the full ~50KB tool schema; loading 10 characters via 10 separate `_get` calls vs. one bulk-get is the difference between 500KB and 50KB of overhead. The bulk-get tool caps at 100 ids per call. (Locations / lore / events don't yet have bulk-get equivalents — for those, the loop is currently unavoidable but try to keep the count small.)

## Anti-patterns

- **Creating a duplicate.** If the user says "add Sasha to *Curses and Currents*", check `stos_characters_list` first — Sasha may already exist under another title and just need a `stos_worldbuilding_link`. If the user already knows there are duplicates, route them to the **`character-merge`** skill — it has a server-side duplicate finder that's dramatically cheaper than reviewing pairs in conversation.
- **Saving a stub.** A character record with only `character_name` is technically valid but useless. Draft the full record before saving.
- **Looping `_get` to compare multiple records.** Use `stos_characters_get_many` for ≥2 characters. For dedup work, use `character-merge`.
- **Sending non-canonical enum values.** See the table above. The save will appear to succeed but the field will be dropped silently.
