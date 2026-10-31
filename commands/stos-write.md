---
description: Draft or expand a scene or chapter
argument-hint: <scene description or chapter slug>
allowed-tools: [Bash, Read, Write]
---

# Draft / Expand

Draft or expand the following: $ARGUMENTS

Follow the `chapter-drafting` skill. Pull relevant story-bible context first (book, characters, locations, lore) via `stos_titles_get`, `stos_characters_list`, etc., and read existing prose with `stos_scenes_get_content`, then draft in the pen name's voice. After the user reviews, save the prose with `stos_scenes_set_content` (a new chapter already has an empty Scene 1 to write into), and check each save result as the `chapter-drafting` skill describes (tell the author if earlier chapter text was saved to Version history).
