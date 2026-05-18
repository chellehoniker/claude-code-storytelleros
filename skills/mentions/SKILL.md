---
name: mentions
description: Use when the user wants to link scene prose to Codex entries (characters, locations, lore) — adding @-mentions explicitly, auto-detecting which characters appear in a scene, or asking "where does X show up in this book?" Triggers on "tag Quinn in this scene", "auto-detect characters", "find every scene the lighthouse appears in", "which characters are in chapter 3", "re-run mention detection".
---

# Mentions

## What this is

Every scene carries a list of **mentions** — typed pointers from the scene to a Codex entry (character / location / lore / event). Two sources:

| Source         | How it's created                                                    | When the detector touches it |
|----------------|---------------------------------------------------------------------|------------------------------|
| `manual`       | The writer placed an @-chip in the editor, or this skill called it. | **Never** — manual wins.     |
| `auto-detected`| The server alias-matched the scene's prose against the Codex.       | Rewritten every detect run.  |

The detector also **skips any entity that already has a manual mention in the same scene** — so a character explicitly tagged once stays tagged exactly once, even if the prose says their name twenty times.

Aliases drive detection. A character "Quinn O'Hara" with `aliases: ['Quinn', 'Q', 'Captain Quinn']` will match all four spellings; word-boundary regex (so "Quinn" doesn't match "Quinnipiac"). Set aliases via `stos_characters_update({ id, fields: { aliases: [...] } })` (same for locations and lore).

## Flow: list mentions for a scene

```js
stos_mentions_list({ sceneId })
```

Returns each mention with `entity_type`, `entity_id`, `text_position`, `matched_text`, `source`.

## Flow: list every scene a character appears in

```js
stos_mentions_list({ entityId: characterId, entityType: 'character' })
```

Useful for "which scenes does Quinn appear in" or building a per-character timeline.

## Flow: create a manual mention

```js
stos_mentions_create({
  scene_id: '...',
  entity_type: 'character',
  entity_id: '...',
  text_position: 1284,           // optional — char offset into the draft
  matched_text: 'Quinn',         // optional — the literal text the chip points at
})
```

`source` defaults to `manual`. The mention is unique per `(scene_id, entity_type, entity_id, text_position)` so calling twice at the same offset is a no-op.

## Flow: run alias detection for a scene

```js
stos_mentions_detect({ scene_id })
```

Wipes the scene's existing `auto-detected` rows, re-runs alias matching against the scoped Codex (characters in this series + locations/lore in this series or this title), and writes a fresh set. Manual mentions are untouched. Returns `{ scene_id, detected: <count>, mentions: [...] }`.

Call this after the writer edits the scene prose, or backfill across an existing manuscript by looping over every scene in a title.

## Flow: delete a mention

```js
stos_mentions_delete({ id })
```

If the writer removed an @-chip, delete its row by id. To clear all auto-detected rows for a scene, run `stos_mentions_detect` after wiping aliases — easier than batch-deleting.

## When to use this vs. `worldbuilding`

| Situation                                       | Skill           |
|-------------------------------------------------|-----------------|
| Add a new character / location / lore entry     | `worldbuilding` |
| Set or update aliases on an existing entry      | `worldbuilding` |
| Tag an entry inside a scene                     | `mentions`      |
| Re-scan a scene's prose for entity references   | `mentions`      |
| "Where does Quinn appear?"                      | `mentions`      |

## Anti-patterns

- **Forgetting aliases.** If a character is "Quinn O'Hara" in the Codex but the prose calls her "Quinn", the detector won't match unless `aliases` includes `Quinn`. Add the alias once via `stos_characters_update` and re-run detect.
- **Running detect on every keystroke.** The detector wipes and rewrites; debounce ~800ms client-side, or batch on save.
- **Tagging an `event` and expecting auto-detect.** Events have no `aliases` column today. Manual mentions work; auto-detect skips them.
- **Mixing `source` in `stos_mentions_create`.** Leave `source` as the default `manual`. The `auto-detected` source is reserved for the detector; setting it manually muddies the "manual wins" rule.
