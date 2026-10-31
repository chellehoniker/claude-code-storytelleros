---
name: markers
description: Use when the user wants to flag a passage for later attention (plot holes, TODOs, or notes) inline on the manuscript. Triggers on "mark a plot hole in chapter X", "leave a TODO at this passage", "what plot holes are still open in this book", "resolve that note", "show me every open marker in Curses and Currents".
---

# Markers

## What this is

A **marker** is an annotation the writer leaves on a passage in a scene. Markers are stored as **notes**: a note becomes a marker when it carries a `marker_type` and a `scene_id`. There is no separate markers tool; use the `stos_notes_*` tools.

| `marker_type` | When to use it |
|---|---|
| `plot-hole` | "Readers will ask why X." Continuity or logic risks flagged for revision. |
| `todo` | "Come back, research the medieval falconry detail." |
| `note` | "This foreshadows chapter 14." Editorial or craft observations. |

A marker can point at a **character range** inside the scene's prose (`text_position_start` / `text_position_end`, offsets into the text from `stos_scenes_get_content`), or at the whole scene when those are left out. Resolving a marker stamps `resolved_at` instead of deleting it, so the record of what was flagged and fixed stays intact.

## Flow: create a marker

```js
stos_notes_create({
  fields: {
    note_title: 'Quinn knows about the will too early',   // required
    content: 'How does Quinn know about the will before chapter 4?',
    chapter_id: '...',
    scene_id: '...',
    marker_type: 'plot-hole',
    text_position_start: 1280,   // optional
    text_position_end: 1380,     // optional
  },
})
```

Pass the offsets when the writer has a specific passage in mind; leave them out for scene-level commentary.

## Flow: list markers

`stos_notes_list` filters by `chapterId` only. It does not filter by scene, marker type, or open/resolved, so do that yourself on the returned notes:

```js
stos_notes_list({ chapterId })   // then keep notes where scene_id matches, marker_type is set, and resolved_at is null
```

For a whole book ("what's still flagged across my manuscript"), list the chapters with `stos_chapters_list({ bookId: titleId })` and call `stos_notes_list({ chapterId })` for each. Keep notes with a `marker_type` and no `resolved_at`. Group by chapter and scene when you report.

## Flow: resolve or reopen

```js
stos_notes_update({ id, fields: { resolved_at: new Date().toISOString() } })   // resolve
stos_notes_update({ id, fields: { resolved_at: null } })                         // reopen
```

## Flow: delete (only when you really mean it)

```js
stos_notes_delete({ id })
```

Use this when the marker was created in error or the writer wants it gone entirely. The normal "I'm done with this concern" path is resolve, not delete.

## When to use this vs. `mentions`

| Situation | Skill |
|---|---|
| "Quinn appears in this scene" | `mentions` |
| "Readers will ask why Quinn knows this here" | `markers` |
| "Tag the Codex entity in the prose" | `mentions` |
| "Leave a writer-note on this passage" | `markers` |
| Tracking process / open questions | `markers` |
| Tracking data / who appears where | `mentions` |

## Anti-patterns

- **Looking for `stos_markers_*` tools.** They were folded into notes; use `stos_notes_*`.
- **Deleting instead of resolving.** Resolution keeps the history of what was flagged.
- **Stuffing structural beats into markers.** Story-structure details (beat, act, synopsis) belong on the scene or chapter record. Use `stos_scenes_update` or `stos_chapters_update` for those fields.
- **A chapter-level note dressed up as a marker.** If the writer's note is about the whole chapter, create a plain note with `chapter_id` and no `scene_id` or `marker_type`.
