---
description: Generate a full story bible for a title — looped, per-entity saves
argument-hint: <title name or id>
allowed-tools: [Bash, Read, Write]
---

# Story Bible

Generate a full story bible for: $ARGUMENTS

Follow the `story-bible` skill. Confirm pen-name + title scope first, then loop:

1. Draft each **character** as a complete record, save with `stos_characters_create` (one call per character — never batch).
2. Draft each **location**, save with `stos_locations_create`.
3. Draft each **story-timeline event**, save with `stos_events_create`.
4. Draft each **lore entry**, save with `stos_lore_create`.
5. Wire associations with `stos_worldbuilding_link` (also one at a time).

Show counts at the end (characters created, locations created, etc.) so the user can verify nothing was truncated.
