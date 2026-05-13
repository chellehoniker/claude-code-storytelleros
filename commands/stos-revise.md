---
description: Revise a chapter using dev-edit-style feedback
argument-hint: <chapter (id, number, or title)> -- <feedback>
allowed-tools: [Bash, Read, Write]
---

# Revise Chapter

Revise this chapter with the given feedback: $ARGUMENTS

Follow the `chapter-drafting` skill in revise mode:

1. Resolve the pen name + title scope (`stos_pen_names_list`, `stos_titles_list`).
2. Find the chapter (`stos_chapters_list`).
3. Pull the persona (`stos_pen_names_get`) and worldbuilding context (`stos_characters_list`, `stos_locations_list`, `stos_lore_list`).
4. Read the current chapter contents — get its `manuscript_storage_path` from `stos_chapters_get` and read the file via storage if needed.
5. Apply the feedback while preserving voice and continuity. Show the revised text to the user.
6. On approval, `stos_chapters_update({ id, fields: { ... } })`.
