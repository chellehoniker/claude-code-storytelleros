---
name: character-merge
description: Use when the user wants to merge, deduplicate, consolidate, or clean up duplicate CHARACTERS specifically. Triggers on "merge characters", "dedupe characters", "I have two records for [name]", "consolidate Drake and Drake Kingston", "find duplicate characters", "clean up my characters". If the user mentions locations or lore in the same breath, use `worldbuilding-merge` instead — it covers all three kinds with the same workflow.
---

# Character merge — cheap, server-side first

## Why this skill exists

A naive merge flow ("list all characters, get each one, compare in conversation") loads the full text of every character into the chat window. With ~50 characters at ~4 KB each, that's 200 KB of context PLUS the full MCP tool schema replayed on every turn. One author burned ~90% of his Claude Pro budget in a single merge pass before this skill existed.

**Two server-side tools do the expensive work for you. Use them.**

## The cheap workflow

1. **`stos_characters_find_duplicates({ penNameId?, seriesId?, minScore? })`** — call this FIRST. The server normalizes names, intersects aliases, and clusters candidates with similarity scores. Returns only `id`, `name`, `aliases`, `score`, `matchedOn` per candidate — small enough to display dozens of groups in one turn.

2. **Show the candidate groups to the user.** Let them confirm which groups to merge. They may want to merge some pairs but not others (e.g. "Drake" and "Drake Kingston" are the same person, but "Drake" and "Drake Senior" are father and son).

3. **`stos_characters_get_many({ ids: [...] })`** — for the candidates the user confirmed, bulk-fetch full records in ONE call. Cap is 100 ids per call. This is the only point where heavy character text enters the conversation.

4. **Pick a "winner" record.** Usually the one with the most complete fields (longest background, most aliases, has an image). Show the user your pick and let them override.

5. **Consolidate fields onto the winner.** Combine `aliases` into a single deduplicated array. Pick the best version of each text field (or concatenate if both have unique content). Call `stos_characters_update({ id: winnerId, fields: { ... } })`.

6. **Preserve the loser's links before deleting.** The duplicate may be referenced by other records — chapters, scenes, events, lore. Before deleting, you can:
   - Call `stos_worldbuilding_link({ targetType: 'character', targetId: winnerId, ... })` for each link the loser had — but the API doesn't currently expose a "list all links for character X" endpoint, so this step is limited. Tell the user: "I'll delete the duplicate, but any chapter/scene that was linked to it specifically will lose that link. Worth re-running the codex auto-detector after." (See `worldbuilding` skill for re-detection.)

7. **`stos_characters_delete({ id: loserId })`** — drop the duplicate.

## What NOT to do

- ❌ **Do not call `stos_characters_get` in a loop.** That's the expensive pattern this skill exists to replace. If you find yourself writing `for (id of ids) get(id)`, stop and use `stos_characters_get_many` instead.
- ❌ **Do not call `stos_characters_list` and then `get` on every result.** `find_duplicates` already returns only the slim fields you need to start.
- ❌ **Do not skip the user-confirm step.** Auto-merging without showing the candidate groups is dangerous — false positives (parent/child, mentor/protégé with similar names) will destroy real records.
- ❌ **Do not delete the loser without consolidating fields first.** Once it's gone, any unique text it had (an aliasing pattern, a backstory paragraph) is gone with it.

## Tuning `minScore`

Default is `0.7`. Lower it to catch more candidates (more false positives, more user review). Raise it to be conservative (may miss subtle dupes).

- `1.0` — exact-name matches only
- `0.95` — exact name OR shared alias
- `0.85` — substring overlap ("Drake" ∈ "Drake Kingston")
- `0.7` (default) — token overlap on full name + aliases
- `0.5` — very permissive; expect many false positives

## Example dialogue

> **User:** I have duplicates in my characters. Can you clean them up?
>
> **You:** I'll look for candidates first. *(call `stos_characters_find_duplicates({ penNameId: 'xyz' })`)*
>
> Found 4 candidate groups:
> - **Drake** + **Drake Kingston** (shared_alias, score 0.95)
> - **Sasha** + **Sasha M.** (substring_overlap, 0.85)
> - **Kat** + **Kat Pierson** + **Katherine Pierson** (shared_alias, 0.95)
> - **Eric** + **Eric the Baby Dragon** (substring_overlap, 0.85)
>
> Do you want me to merge any of these? Note that the Kat group has three records and the Eric pair might be intentional if one is a younger version.
>
> **User:** Merge the Drake pair and the Kat group. Eric is fine — separate.
>
> **You:** *(call `stos_characters_get_many({ ids: [drake1, drake2, kat1, kat2, kat3] })` — ONE call for all five.)*
>
> Drake Kingston has the longer background and the dragon-knight backstory. Drake is just a name + age. I'll merge Drake into Drake Kingston (keeping Drake as an alias).
>
> For the Kat group, Katherine Pierson has the most detail. I'll fold Kat + Kat Pierson into Katherine Pierson, with both shorter names as aliases.
>
> *(call `stos_characters_update` for each winner, then `stos_characters_delete` for the three losers.)*
>
> Done. 2 records merged into Drake Kingston, 3 records merged into Katherine Pierson. I'd recommend re-running the codex auto-detector on your scenes so any references that were pointing at the deleted records get re-linked to the winners.

## Forward-looking

`stos_locations_find_duplicates` + `stos_locations_get_many` and `stos_lore_find_duplicates` + `stos_lore_get_many` now exist — the sibling `worldbuilding-merge` skill covers location and lore dedup using the same workflow shape as this skill. Use this skill for character-only sessions; use `worldbuilding-merge` when the user wants to clean up multiple worldbuilding kinds in one pass.

Events do not yet have a duplicate-finder. If the user asks for event dedup, fall back to `stos_events_list` and compare event_name + event_date in the conversation — and tell them we'll have a proper tool soon.
