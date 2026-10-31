---
description: Revise a chapter using editorial feedback
argument-hint: <chapter (id, number, or title)> -- <feedback>
allowed-tools: [Bash, Read, Write]
---

# Revise Chapter

Revise this chapter with the given feedback: $ARGUMENTS

Follow the `chapter-drafting` skill in revise mode:

1. Resolve the pen name + title scope (`stos_pen_names_list`, `stos_titles_list`).
2. Find the chapter (`stos_chapters_list`).
3. Pull the persona (`stos_pen_names_get`) and worldbuilding context (`stos_characters_list`, `stos_locations_list`, `stos_lore_list`).
4. Read the current chapter text: `stos_scenes_list({ chapterId })`, then `stos_scenes_get_content({ id })` for each scene.
5. Apply the feedback while preserving voice and continuity. Show the revised text to the user.
6. On approval, save each changed scene with `stos_scenes_set_content({ id, content })`. Chapter records don't hold the text, so `stos_chapters_update` can't save it.
