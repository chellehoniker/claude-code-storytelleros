---
name: pen-names
description: Use when the user mentions a specific pen name, has multiple pen names, or asks to switch between author identities. Triggers on phrases like "draft under my [pen name]", "switch to [pen name]", "log time for [pen name]", "my other pen name", "list my pen names".
---

# Switching Between Pen Names

## Why this matters

Authors who write under multiple pen names rely on StorytellerOS' pen-name separation: each pen name has its own books library, characters, locations, lore, calendar, finance ledger, tasks, and brand voice guides. The web app has a top-right dropdown to switch between them.

In the plugin, every profile-scoped tool (`stos_books_list`, `stos_characters_list`, `stos_tasks_create`, `stos_time_tracking_start`, `stos_articles_list`, etc.) accepts an optional pen-name selector. **Without one, calls go to the user's active / primary pen name.**

<!-- TODO: confirm parameter name once Phase 1A REST routes land — current intent per the plan is either an `X-Profile-Id` header forwarded by the MCP layer or a `penNameId` argument on each tool. Update this skill once the surface is finalized. -->

## When to use this skill

The user mentions a pen name by name, OR the user has multiple pen names, OR you suspect you're addressing the wrong identity (e.g., `stos_books_list` returns an empty list and the user said they have books drafted).

## Flow

### 1. Discover their pen names

Call `stos_pen_names_list` once. It returns:

```json
{
  "penNames": [
    { "id": "pn_abc123...", "name": "chelle", "isPrimary": true },
    { "id": "pn_def456...", "name": "Indie Annie", "isPrimary": false }
  ]
}
```

`isPrimary: true` is the user's default profile (the one used when no selector is passed).

### 2. Match the user's intent to a pen name

If the user said "draft a chapter for Indie Annie" and you see a pen name "Indie Annie," that's the match. If the match is ambiguous, ask the user to confirm: "I see two pen names — chelle and Indie Annie. Which one should this go under?"

### 3. Pass the pen-name selector on every subsequent profile-scoped call

```js
// Before pen-name switch
stos_books_list({})  // → primary pen name's books

// After
stos_books_list({ penNameId: "pn_def456..." })  // → Indie Annie's books
stos_chapters_update({ penNameId: "pn_def456...", chapterId, content })
stos_tasks_create({ penNameId: "pn_def456...", title: "..." })
```

The selector doesn't carry across calls automatically. If the user is working in a single pen name for a multi-step flow (drafting several chapters, planning a series), include it on EVERY tool call in the sequence.

### 4. Brand voice and story bible change per pen name

Story bible reads (`stos_characters_list`, `stos_locations_list`, `stos_lore_list`) return different data per pen name. ALWAYS scope them to the matching pen name before drafting prose — a cozy-mystery pen name's voice and world is different from a thriller pen name's.

## Common mistakes to avoid

- **Forgetting the selector mid-sequence.** If the user has been working under a non-primary pen name for the last several tool calls, keep using its id. Don't silently revert to primary.
- **Calling `stos_pen_names_list` multiple times.** It's the same data each time — call it once at the start of the conversation when needed and remember the IDs.
- **Treating an empty `stos_books_list` as "user has nothing."** First check: did you scope to the right pen name? Their books might live under a different pen name.
- **Mixing pen names in a single record.** A character belongs to one pen name. A book belongs to one pen name. Don't try to attach a character from pen name A to a book under pen name B.

## Sample exchange

**User:** "Draft chapter 3 of my Indie Annie cozy. The previous chapter ended with the cat knocking the urn over."

**You:**
1. Call `stos_pen_names_list` — find "Indie Annie", get id
2. Call `stos_books_list({ penNameId })` — find the active cozy and book id
3. Call `stos_chapters_list({ penNameId, bookId })` — find chapter 3 (or the next slot)
4. Call `stos_characters_list({ penNameId, bookId })` and `stos_locations_list({ penNameId, bookId })` — load story-bible context
5. Draft the chapter in the cozy voice, given the urn cliffhanger
6. Show the draft to the user, then on approval `stos_chapters_update({ penNameId, chapterId, content })`

If the selector is omitted on any of those calls, it'd go to the primary pen name — wrong identity, wrong story bible, wrong voice.
