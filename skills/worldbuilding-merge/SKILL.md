---
name: worldbuilding-merge
description: Use when the user wants to merge, deduplicate, consolidate, or clean up duplicate CHARACTERS, LOCATIONS, or LORE in their story bible. Triggers on "merge locations", "dedupe lore", "I have two records for [place/thing]", "consolidate these characters", "find duplicate locations", "clean up my worldbuilding". For character-only merges, the older `character-merge` skill is still valid — this skill is the broader umbrella covering all three kinds.
---

# Worldbuilding merge — cheap, server-side first

## Why this skill exists

A naive merge flow ("list all X, get each one, compare in conversation") loads the full text of every record into the chat window. With ~50 characters at ~4 KB each, that's 200 KB of context PLUS the full MCP tool schema replayed on every turn. One author burned ~90% of his Claude Pro budget in a single character-merge pass before the server-side tools existed. The same trap applies to locations and lore — locations with long `description` + `key_features`, lore entries with long `rules_and_mechanics` + `origin_and_history`.

**Three pairs of server-side tools do the expensive work for you. Use them.**

| Entity | Duplicate finder | Bulk get |
|---|---|---|
| Characters | `stos_characters_find_duplicates({ penNameId?, seriesId?, minScore? })` | `stos_characters_get_many({ ids })` |
| Locations | `stos_locations_find_duplicates({ minScore? })` | `stos_locations_get_many({ ids })` |
| Lore | `stos_lore_find_duplicates({ bookId?, minScore? })` | `stos_lore_get_many({ ids })` |

(Events do not yet have a duplicate-finder. If the user asks for event dedup, fall back to `stos_events_list` and compare event_name + event_date in the conversation — and tell them we'll have a proper tool soon.)

## The cheap workflow (same for all three kinds)

1. **Call `_find_duplicates` FIRST.** The server normalizes names, intersects aliases, and clusters candidates with similarity scores. Returns only `id`, `name`, `aliases`, `score`, `matchedOn` per candidate. Small payload — fits dozens of groups in one turn.

2. **Show the candidate groups to the user.** Let them confirm which groups to merge. Some clusters will be false positives — sibling characters with similar names, two distinct locations with shared aliases ("the keep" can mean different keeps in different books), parent and child lore entries.

3. **Call `_get_many({ ids: [...] })` for confirmed candidates only.** Bulk-fetch the full records in ONE round-trip. Cap is 100 ids per call. This is the only step where heavy text enters the conversation.

4. **Pick the "winner" record.** Usually the one with the most complete fields. Show the user your pick and let them override.

5. **Consolidate fields onto the winner.** Combine `aliases` into a single deduplicated array. Pick the best version of each text field (or concatenate if both contain unique content). Call `stos_{kind}_update({ id: winnerId, fields: { ... } })`.

6. **Heads-up to the user about junction links.** Worldbuilding-link junctions (chapter ↔ location, scene ↔ character, series ↔ lore, etc.) reference the loser by id. There is no "list all links for X" endpoint today, so when you call `stos_{kind}_delete({ id: loserId })`, any link pointing at the loser will cascade-delete with it. Tell the user: "Once I delete the duplicate, any specific chapter/scene that was linked to it will lose that link. I'd recommend re-running the codex auto-detector on your scenes afterwards to re-establish references." (See the `worldbuilding` skill for re-detection.)

7. **Delete the loser** with `stos_{kind}_delete({ id: loserId })`.

## What NOT to do

- ❌ **Do not call `stos_{kind}_get` in a loop.** That's the expensive pattern this skill exists to replace. If you find yourself writing `for (id of ids) get(id)`, stop and use `_get_many` instead.
- ❌ **Do not call `stos_{kind}_list` and then `_get` on every result.** `_find_duplicates` already returns only the slim fields you need to start.
- ❌ **Do not skip the user-confirm step.** Auto-merging without showing the candidate groups is dangerous — false positives (parent/child characters, distinct locations with shared aliases, magic-system parent/child lore) will destroy real records.
- ❌ **Do not delete the loser without consolidating fields first.** Once it's gone, any unique text it had is gone with it.
- ❌ **Do not run a cross-kind merge.** A location is not a character. The user may sometimes have miscategorized — a "character" that's actually a faction/group, or a "lore entry" about a specific place. In those cases, the right move is `delete from wrong kind` + `create in right kind`, not a merge — and the user should drive that decision.

## Tuning `minScore`

Default is `0.7`. Lower it to catch more candidates (more false positives, more user review). Raise it to be conservative (may miss subtle dupes).

- `1.0` — exact-name matches only
- `0.95` — exact name OR shared alias
- `0.85` — substring overlap ("Drake" ∈ "Drake Kingston", "the Keep" ∈ "the High Keep")
- `0.7` (default) — token overlap on full name + aliases
- `0.5` — very permissive; expect many false positives

## Scope filters per kind

- **Characters:** filter by `seriesId` (only consider characters in one series) or `penNameId` (resolved via series lookup). Useful when an author has multiple series and only wants to dedupe within one.
- **Locations:** no scope filter today — server returns all the user's locations. Filter client-side if the user wants a narrower scope.
- **Lore:** filter by `bookId` to only compare lore for one book. Useful when an author has both series-wide lore (shared cosmology) and book-specific lore.

## Example dialogue — location merge

> **User:** I think I have duplicate locations in my worldbuilding. Can you check?
>
> **You:** *(call `stos_locations_find_duplicates({})`)*
>
> Found 3 candidate groups:
> - **The Keep** + **High Keep** + **The High Keep** (shared_alias, score 0.95)
> - **Crystal Caverns** + **Crystal Cave** (substring_overlap, 0.85)
> - **Boerne** + **Boerne, Texas** (substring_overlap, 0.85)
>
> Want to merge any of these? Note that "Crystal Caverns" and "Crystal Cave" might be intentional if one is a smaller area within the other.
>
> **User:** Merge the Keep group into one record. The crystal ones are different. The Boerne pair can merge.
>
> **You:** *(call `stos_locations_get_many({ ids: [keep1, keep2, keep3, boerne1, boerne2] })` — ONE call for all five.)*
>
> "The High Keep" has the longest description and the most detailed key_features. I'll merge "The Keep" + "High Keep" into "The High Keep" (keeping both shorter names as aliases). For Boerne, "Boerne, Texas" has more development_notes and the climate field filled in — I'll fold "Boerne" into it.
>
> *(call `stos_locations_update` for each winner, then `stos_locations_delete` for the three losers.)*
>
> Done. 3 records merged into "The High Keep", 2 merged into "Boerne, Texas". I'd recommend re-running the codex auto-detector on your scenes so any references that were pointing at the deleted records get re-linked to the winners.

## Relationship to other skills

- **`character-merge`** — older single-purpose skill, characters only. Still works fine for character-only sessions; this skill is the broader umbrella when the user wants to clean up multiple worldbuilding kinds.
- **`worldbuilding`** — single-entry add/edit. Not for dedup. If the user is editing fields on a record that already exists, that's `worldbuilding`, not this skill.
- **`story-bible`** — full extraction. After a story-bible run, there may be near-duplicates (alternate spellings, same character introduced under a nickname). Running this skill afterwards is the cleanup pass.
