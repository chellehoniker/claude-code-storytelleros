---
description: Search across titles, characters, locations, and lore
argument-hint: <query>
allowed-tools: [Bash, Read, Write]
---

# Search

Search StorytellerOS for: $ARGUMENTS

Query the relevant resources in parallel (`stos_titles_list`, `stos_characters_list`, `stos_locations_list`, `stos_lore_list`), filtered to the user's active pen name where applicable. Return a grouped, scannable summary with IDs so the user can pick something to drill into.
