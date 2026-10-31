---
name: chapter-drafting
description: Use when the user wants to draft, rewrite, or revise a chapter or scene. Triggers on "draft chapter [N]", "write scene [name]", "rewrite this chapter", "continue from where I left off", "draft the next chapter of [title]". Pulls pen-name voice + story-bible context before writing.
---

# Chapter Drafting

## When to use

The user wants Claude to produce prose for a specific chapter or scene of a specific title. This is the deepest writing flow — Claude reads the pen name's persona, the title's existing chapters, the relevant characters/locations/lore, and then drafts in the right voice.

## Flow

### 1. Resolve scope

If the user named a pen name and a title, look them up:

```js
stos_pen_names_list()          // find the matching id
stos_titles_list({ penNameId })  // find the matching title id
```

If either is ambiguous, ask the user to clarify. Don't guess.

### 2. Read voice + bible

```js
stos_pen_names_get({ id: penNameId })   // persona, prose_guide, prose_sample, words_to_use, words_to_avoid
stos_chapters_list({ bookId: titleId }) // existing chapters for continuity
stos_scenes_list({ chapterId })         // the chapter's scenes, in order
stos_scenes_get_content({ id: sceneId }) // a scene's FULL prose (draft_snippet is only a preview)
stos_characters_list({ penNameId })     // who can appear
stos_locations_list({ penNameId })      // where it can happen
stos_lore_list({ penNameId })           // mechanics / rules to respect
```

### 3. Draft

Write the chapter in the conversation. Honor:

- The persona's `prose_guide` (voice / tense / POV).
- `words_to_use` and `words_to_avoid` if present.
- Continuity from the prior chapter and from any scene beats already saved.
- Character voice patterns (lexicon_and_quirks, speech_patterns) and physical descriptions.
- Location features (climate, key_features).
- Lore rules (rules_and_mechanics) — never contradict.

Show the draft to the user. **Wait for approval** before saving.

### 4. Save

Prose lives on **scenes**, and it is saved with `stos_scenes_set_content({ id, content })`. Chapter and scene records hold details like title, synopsis and beat, not the text, so `stos_chapters_update` and `stos_scenes_update` cannot save prose.

- **Existing chapter:** write each changed scene with `stos_scenes_set_content`. The chapter text and word count rebuild from its scenes, and the app keeps a version the author can restore.
- **New chapter:** `stos_chapters_create({ fields: { book_id: titleId, chapter_number, chapter_title } })`. It already has an empty Scene 1: get it with `stos_scenes_list({ chapterId })` and write to it. Don't create a second Scene 1.
- **More scenes:** `stos_chapters_insert_scene({ id: chapterId, order })`, then `stos_scenes_set_content` on the new scene. If that answers paused (409 `STRUCTURE_OPS_PAUSED`), create the scene with `stos_scenes_create({ fields: { book_id, chapter_id, scene_title, scene_order } })` instead.

#### Check every save result

After each `stos_scenes_set_content`, read `chapter_synced`, `chapter_sync_reason` and `chapter_snapshot_version_id`:

- **`chapter_sync_reason: 'adopted_scenes'`:** the chapter had text its scenes didn't (an edit made in the app, or a restored version). That earlier chapter text was saved first, in the chapter's version history, as a version named "Before assistant update" (its id is `chapter_snapshot_version_id`); then the chapter was rebuilt from the scenes. Always tell the author, for example: "Your chapter had changes I hadn't seen, so I saved the earlier text first. To get it back, open the chapter in the Writing Studio, open Version history, and restore the version called Before assistant update."
- **`chapter_synced: false`:** the scene is saved, but the chapter text was left as it was for now (for example, another scene in the chapter has no text yet). It catches up on a later save. Mention it if the author is about to read the chapter.
- **409 `CHAPTER_NOT_REBUILDABLE`:** nothing was written. Wait a few seconds and retry the same call once. If it fails again, stop and tell the author the save didn't go through and their text is unchanged.
- `version_recorded: false` only means no history snapshot was taken; the prose is still saved.

If the author has a whole manuscript file to bring in, use the `manuscript-import` skill instead of saving chapter by chapter.

## Revise mode

When the user asks to revise a chapter with feedback (the `/stos-revise` command is this), pull the existing text, apply the feedback, and show the revised draft. Don't blindly accept the feedback's wording — apply it in the pen name's voice.

## Anti-patterns

- **Drafting without reading the bible.** A chapter is only as good as the continuity it preserves. Always pull `stos_characters_list` and `stos_locations_list` first.
- **Saving without showing the user.** Always show prose for approval before `stos_scenes_set_content`.
- **Saving prose with `stos_chapters_update` or `stos_scenes_update`.** Neither holds the text. Use `stos_scenes_set_content`.
- **Forgetting the pen-name scope.** A chapter drafted in the wrong persona reads off-brand. Pass `penNameId` on every call in the multi-step flow.
- **Skipping the persona's `prose_sample`.** It's the highest-fidelity voice anchor available.
