---
name: story-bible
description: Use when the user wants to generate, extract, or build out a full story bible for a title. Triggers on "generate a story bible", "build out the world of [title]", "extract characters from this manuscript", "set up the worldbuilding for [series]", "what characters appear in [title]". Loops per-entity — no bulk shortcuts.
---

# Story Bible

## What this is

A "story bible" in StorytellerOS is the union of four worldbuilding tables: **characters**, **locations**, **events** (story timeline, not calendar), and **lore**. Plus the junctions that wire them into titles, chapters, and scenes.

When the user asks Claude to build a bible, the work is producing structured records for each entry — every character with their core_desire, core_fear, character_arc, etc.; every location with key_features, climate; every event with act and importance; every lore entry with rules_and_mechanics and origin_and_history.

## The no-shortcuts rule

**One record at a time.** Even if Claude could in principle assemble a JSON array of 30 characters and POST them in one call, don't — large manuscripts trip context windows mid-batch and entries get truncated. Loop instead:

```
for each character in the manuscript:
  stos_characters_create({ fields: { character_name, role_in_story, core_desire, core_fear, ... } })
```

Same for locations, events, lore. The user explicitly asked for the fullest possible output and no shortcuts; this is enforced at the skill level, not by the tools.

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

1. Draft the full record in the conversation (name, role, age, occupation, pronouns, gender, life_status, background, physical_description, personality, pov_status, dialogue_style, core_desire, core_fear, goals_and_motivations, strengths, flaws, internal_conflict, external_conflict, character_arc, backstory, key_relationships, lexicon_and_quirks, emotional_tells, speech_patterns, internal_voice, sample_scene).
2. Show it briefly to the user for sanity-check on the first 2–3 entries; then proceed.
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

### 8. Report

End with counts:

> "Saved 14 characters, 7 locations, 12 events, 5 lore entries, and 38 associations to *Curses and Currents*."

If any single `_create` call failed mid-loop, surface the failure with the entry name so the user can retry that specific record.

## Anti-patterns

- **Batching multiple entries into one tool call.** Hard no. The whole point of this skill is the per-entity loop.
- **Skipping fields to save time.** If you have the information, save it. Truncating a character's `backstory` to a one-liner defeats the bible.
- **Forgetting associations.** A character that isn't linked to any chapter or scene is dead weight — wire every connection the source implies.
- **Drafting under the wrong pen name.** Same continuity hazard as `chapter-drafting`.

## Tip for very large manuscripts

If the manuscript is >200k words, ask the user whether they want to process it chapter-by-chapter rather than as a single pass. STOS already has `/api/extract-manuscript-bible/run` for that — the in-app version of this skill — but the CoWork story-bible flow does it conversationally.
