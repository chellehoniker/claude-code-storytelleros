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
stos_scenes_list({ chapterId })         // scene-level beats if any
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

If the chapter already exists, `stos_chapters_update({ id, fields })`. If it doesn't, `stos_chapters_create({ fields })` first.

For scene-level revisions, use `stos_scenes_update` / `stos_scenes_create`.

If the user uploaded a new manuscript file as the source of truth, register it with `stos_title_manuscripts_create` and (if they want) `stos_title_manuscripts_set_active`.

## Revise mode

When the user asks to revise a chapter with feedback (the `/stos-revise` command is this), pull the existing text, apply the feedback, and show the revised draft. Don't blindly accept the feedback's wording — apply it in the pen name's voice.

## Anti-patterns

- **Drafting without reading the bible.** A chapter is only as good as the continuity it preserves. Always pull `stos_characters_list` and `stos_locations_list` first.
- **Saving without showing the user.** Always show prose for approval before `stos_chapters_update`.
- **Forgetting the pen-name scope.** A chapter drafted in the wrong persona reads off-brand. Pass `penNameId` on every call in the multi-step flow.
- **Skipping the persona's `prose_sample`.** It's the highest-fidelity voice anchor available.
