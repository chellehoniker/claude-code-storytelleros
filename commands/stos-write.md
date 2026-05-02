---
description: Draft or expand a scene or chapter
argument-hint: <scene description or chapter slug>
allowed-tools: [Bash, Read, Write]
---

# Draft / Expand

Draft or expand the following: $ARGUMENTS

Pull relevant story-bible context first (book, characters, locations, lore) via `stos_books_get`, `stos_characters_list`, etc., then draft in the pen name's voice. Save the result with `stos_chapters_update` or `stos_scenes_update` after the user reviews.
