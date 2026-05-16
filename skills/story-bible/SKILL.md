---
name: story-bible
description: Use when the user wants to generate, extract, or build out a full story bible for a title. Triggers on "generate a story bible", "build out the world of [title]", "extract characters from this manuscript", "set up the worldbuilding for [series]", "what characters appear in [title]". Routes uploaded manuscripts through the server-side extractor; loops per-entity only for chat-sourced bibles.
---

# Story Bible

## What this is

A "story bible" in StorytellerOS is the union of four worldbuilding tables: **characters**, **locations**, **events** (story timeline, not calendar), and **lore**. Plus the junctions that wire them into titles, chapters, and scenes.

When the user asks Claude to build a bible, the work is producing structured records for each entry — every character with their core_desire, core_fear, character_arc, etc.; every location with key_features, climate; every event with act and importance; every lore entry with rules_and_mechanics and origin_and_history.

## Two paths — pick by source

**The decision is purely about the source of the manuscript text**, not the user's wording.

### Path A — Uploaded manuscript (default, USE THIS WHENEVER POSSIBLE)

If the title already has an uploaded manuscript in STOS — check with `stos_title_manuscripts_list({ titleId })`, or look for `manuscript_storage_path` on the title — kick off the server-side pipeline in **one call**:

```
stos_titles_extract_story_bible({ titleId })
  → { jobId, statusUrl, options, provider }

// Then poll every 5s until terminal:
stos_extraction_jobs_get({ jobId })
  → { status, progress: { stage, pct, detail }, result: { ... } }
```

While polling, surface the progress detail to the user ("Surveying named characters…", "Locations: chunk 2 of 5"). Stop polling when `status === 'completed'` or `'failed'`.

On `'failed'`, the result carries a `terminal_error` block with a ready-to-paste `user_message`. Show it verbatim — don't re-summarise. The classes are:
- `cap` — AI provider monthly cap exhausted
- `billing` — provider account needs credits
- `auth` — saved API key rejected
- `oversize` — single manuscript piece exceeds the provider window (rare with chunking; suggest splitting)
- `empty-output` — provider returned nothing usable; confirm the right file was uploaded
- `rate-limit` — short-window throttle; retry after a minute

On `'completed'`, summarise the per-kind counts from `result.created` / `result.merged` and remind the user that drafts are in Worldbuilding for review.

**Why this is the default**: the per-entity loop in Path B reliably runs out of agent turn budget on a real novel (30+ characters → 8-15 locations → 10-20 events → 5-10 lore entries × ~2k tokens each, plus the loop tool-call overhead). Real users (Deb 2026-05-15) saw the bible "save characters but skip lore/events/locations" because the agent got cut off mid-loop. The server-side path doesn't have that ceiling — it chunks the manuscript per provider, runs each pass with raw prose, and dedupes results before persisting.

After completion, still run the association-wiring step from §7 below (chapter/scene/series links) — the server-side extractor does the core persists, but cross-table junctions are the agent's job.

### Path B — Chat-sourced bible (no uploaded manuscript)

If the source is an outline, synopsis, or premise pasted into chat, the per-entity loop is the only option — there's no file for the server to read. Drop into the loop documented in §3-§7. The "no shortcuts" rule below still applies: one tool call per entry, no batching.

```
for each character in the source:
  stos_characters_create({ fields: { character_name, role_in_story, ... } })
```

Same for locations, events, lore.

## The no-shortcuts rule (Path B only)

**One record at a time.** Even if Claude could in principle assemble a JSON array of 30 characters and POST them in one call, don't — large sources trip context windows mid-batch and entries get truncated. Loop instead. The user explicitly asked for the fullest possible output and no shortcuts; this is enforced at the skill level, not by the tools.

## Canonical field values — send these exactly

The Story Bible tables have strict CHECK constraints. Sending anything other than the canonical value below silently drops the field — the rest of the record saves, but you'll have wasted tokens and the user will see a blank column in the UI. **Map synonyms to the canonical value before calling the tool. If a value can't be mapped, omit the field entirely — never invent, never stash the original somewhere else.**

| Field | Canonical values | Synonym → canonical |
|---|---|---|
| `characters.role_in_story` | `protagonist`, `antagonist`, `supporting`, `minor`, `love-interest`, `other` | hero / heroine / lead / main / MC → **protagonist** • villain / big-bad / rival → **antagonist** • deuteragonist / mentor / sidekick / ally / foil / friend / family → **supporting** • love interest / romantic interest / romance → **love-interest** • background / extra / NPC / cameo / walk-on → **minor** |
| `characters.life_status` | `alive`, `dead`, `undead`, `missing`, `unknown` | deceased / killed → **dead** • living / alive-and-well → **alive** • absent / lost / disappeared → **missing** • ghost / vampire / zombie → **undead** |
| `characters.pov_status` | `pov-character`, `non-pov` | any POV / occasional-POV / multi-POV / "yes" / "true" / "POV" → **pov-character** • everything else (or "no" / "false") → **non-pov** |
| `events.act` | `act-one`, `act-two`, `act-three`, `act-four` | 1 / I / "Act 1" / "Act I" → **act-one** (and 2/II → act-two, etc.) |

**Never send `custom_fields`** on any Cowork story-bible create call. The StorytellerOS UI does not render those keys for Cowork-completed records — the data lands invisible. Stick to the named columns listed in each tool's writable-fields description.

## Flow

### 1. Scope

Find the pen name and the title:

```js
stos_pen_names_list()
stos_titles_list({ penNameId })
```

Confirm scope with the user before running a long loop. ("I'll build the bible for *Curses and Currents* under your Indie Annie pen name — that's 8 chapters, expect ~15 characters, ~8 locations. OK to proceed?")

### 2. Sources of truth

The user may give Claude one of:

- An uploaded manuscript (read it via the title's `word_doc_storage_path` or the latest `book_manuscripts` revision).
- An outline / synopsis pasted in chat.
- The current chapters / scenes already in STOS (`stos_chapters_list`, `stos_scenes_list`).
- Whole-cloth generation from a premise.

Read whatever they provide before drafting entries.

### 3. Loop: characters

For each character mentioned in the source:

1. Assemble the full record (name, role, age, occupation, pronouns, gender, life_status, background, physical_description, personality, pov_status, dialogue_style, core_desire, core_fear, goals_and_motivations, strengths, flaws, internal_conflict, external_conflict, character_arc, backstory, key_relationships, lexicon_and_quirks, emotional_tells, speech_patterns, internal_voice, sample_scene). Apply the canonical-value mapping above before you reach for the tool.
2. Print a one-line summary per entry (`Sasha — protagonist, witch, POV`) so the user can follow along. **Do not paste the full record block inline** — the full record only goes into the tool call. Inline drafting eats the user's token budget without adding value.
3. `stos_characters_create({ fields: { ... } })` — one call per character.
4. Record the returned id so you can link later.

### 4. Loop: locations

Same pattern — one `stos_locations_create` per location with all fields (description, location_type, key_features, climate, coordinates, map_image_url, development_notes).

### 5. Loop: story-timeline events

`stos_events_create` per event — event_name, event_date (in-universe), event_type, description, act, importance, event_order.

### 6. Loop: lore

`stos_lore_create` per lore entry — lore_name, lore_type, description, rules_and_mechanics, examples_and_usage, origin_and_history, contradictions_check, tags.

### 7. Wire associations

For each (entity, target) pair the source implies (e.g. "Sasha appears in chapter 3", "the Crow Court rules govern scene 7"), call `stos_worldbuilding_link` — once per association.

Supported pairs include title↔{character,location,event,lore}, chapter↔{character,event}, scene↔{character,location,event}, series↔{location,lore}, event↔{character,location}, lore↔{character,location}.

### 7a. Scope lore and locations to the series

Characters carry a `series_id` directly, so they show up under the series filter as soon as they're created. **Lore and locations don't** — they live in `lore` and `locations` tables with no series column, and only surface under the series filter when an entry exists in the `series_lore` / `series_locations` junction. Without this step, every lore/location entry orphans visually.

For each lore entry just created:

```
stos_worldbuilding_link({
  entityType: 'series', entityId: <seriesId>,
  targetType: 'lore',   targetId:  <loreId>
})
```

For each location:

```
stos_worldbuilding_link({
  entityType: 'series', entityId: <seriesId>,
  targetType: 'location', targetId: <locationId>
})
```

One call per entry. These run alongside the chapter/scene-level links from Step 7 — they're additive, not a replacement.

### 8. Report

End with counts:

> "Saved 14 characters, 7 locations, 12 events, 5 lore entries, and 38 associations to *Curses and Currents*."

If any single `_create` call failed mid-loop, surface the failure with the entry name so the user can retry that specific record.

## Anti-patterns

- **Batching multiple entries into one tool call.** Hard no. The whole point of this skill is the per-entity loop.
- **Skipping fields to save time.** If you have the information, save it. Truncating a character's `backstory` to a one-liner defeats the bible.
- **Forgetting associations.** A character that isn't linked to any chapter or scene is dead weight — wire every connection the source implies.
- **Forgetting the series-level links from Step 7a.** A lore entry or location without a `series_lore` / `series_locations` row will not appear under the series filter and the user will conclude the bible isn't connected.
- **Sending non-canonical enum values.** Map first (see the *Canonical field values* table); if you can't map, omit the field. Sending "Deuteragonist" or "Act 1" silently drops the field at the DB CHECK constraint and burns retry tokens.
- **Sending `custom_fields`.** Invisible in the StorytellerOS UI for Cowork-completed records. Use only named columns.
- **Pasting full record blocks inline before calling the tool.** The tool call carries the record — printed paragraphs just consume the user's Claude allowance.
- **Drafting under the wrong pen name.** Same continuity hazard as `chapter-drafting`.

## Very large manuscripts

The server-side path (Path A) already chunks the manuscript into provider-fitting pieces and dedupes drafts across chunks, so a 200k-word novel runs in one `stos_titles_extract_story_bible` call. You don't need to pre-split anything. The progress detail surfaced through `stos_extraction_jobs_get` will show the chunk count as it works through the book.

If the server returns `terminal_error.kind === 'oversize'`, that means a *single chapter* exceeds the provider context window — extremely rare. Tell the user, suggest splitting the offending chapter, and don't retry blindly.

Path B's per-entity loop has no chunking — if a user pastes a 200k-word manuscript directly into chat (no upload), stop and ask them to upload it to the title first so you can use Path A. Pasting a novel into chat just to feed the loop wastes their token budget and risks the same mid-loop cutoff.
