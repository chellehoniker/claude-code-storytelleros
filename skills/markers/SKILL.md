---
name: markers
description: Use when the user wants to flag a passage for later attention — plot-holes, TODOs, or notes inline on the manuscript. Triggers on "mark a plot hole in chapter X", "leave a TODO at this passage", "what plot holes are still open in this book", "resolve that note", "show me every open marker in Curses and Currents".
---

# Markers

## What this is

A **marker** is an inline annotation the writer leaves on a passage in a scene. Three types share one table and only differ by render color in the editor:

| Type        | When to use it                                                                |
|-------------|-------------------------------------------------------------------------------|
| `plot-hole` | "Readers will ask why X." Continuity / logic risks flagged for revision.      |
| `todo`      | "Come back, research the medieval falconry detail."                           |
| `note`      | "This foreshadows chapter 14." Editorial / craft observations.                |

Each marker is scoped to a single **scene** and a **character-offset range** inside that scene's prose. Resolving a marker stamps `resolved_at` instead of deleting the row, so the audit trail of what was flagged and fixed stays intact.

## Flow: list open markers in a scene

```js
stos_markers_list({ sceneId })
```

Returns each marker with `marker_type`, `text_position_start`, `text_position_end`, `body`, `resolved_at`. By default returns only **open** markers (resolved_at IS NULL); pass `archived: 'any'` for both, or `archived: 'true'` for resolved-only.

## Flow: list every open marker for the writer

```js
stos_markers_list({})
```

Useful for "what's still flagged across my whole manuscript" — combine with `stos_markers_list({ sceneId })` per scene if you need scene grouping.

## Flow: create a marker

```js
stos_markers_create({
  fields: {
    scene_id: '...',
    marker_type: 'plot-hole',
    text_position_start: 1280,
    text_position_end: 1380,
    body: 'How does Quinn know about the will before chapter 4?',
  },
})
```

`text_position_start` / `text_position_end` are character offsets into the scene draft. Pass them when the writer has a specific passage in mind; omit when the marker is scene-level commentary.

## Flow: resolve / reopen

```js
stos_markers_resolve({ id })              // stamps resolved_at = now()
stos_markers_update({ id, fields: { resolved_at: null } })   // reopens
```

## Flow: delete (only when you really mean it)

```js
stos_markers_delete({ id })
```

Use this when the marker was created in error or the writer wants the audit trail erased. The normal "I'm done with this concern" path is resolve, not delete.

## When to use this vs. `mentions`

| Situation                                                | Skill      |
|----------------------------------------------------------|------------|
| "Quinn appears in this scene"                            | `mentions` |
| "Readers will ask why Quinn knows this here"             | `markers`  |
| "Tag the Codex entity in the prose"                      | `mentions` |
| "Leave a writer-note on this passage"                    | `markers`  |
| Tracking process / open questions                        | `markers`  |
| Tracking data / who appears where                        | `mentions` |

## Anti-patterns

- **Deleting instead of resolving.** Resolution is the documented lifecycle. Deletion erases the audit trail.
- **Stuffing structural beats into markers.** Story-structure metadata (beat name, act, scene synopsis) belongs on the scene record itself, not in marker bodies. Use `stos_scenes_update` for those fields.
- **Markers without scene_id.** The table FKs scenes; there is no "chapter-level" marker by design. If the writer wants a chapter-level note, write it to the chapter's `development_notes` field via `stos_chapters_update`.
