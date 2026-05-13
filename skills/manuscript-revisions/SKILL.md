---
name: manuscript-revisions
description: Use when the user wants to list, upload, or switch between manuscript file revisions for a title. Triggers on "show me my manuscript versions", "I just uploaded a new draft", "switch to the [name] revision", "what manuscripts do I have for [title]".
---

# Manuscript Revisions

## What this is

Each title can carry multiple manuscript file revisions in the `manuscripts` Supabase Storage bucket. Authors using Vellum / Scrivener re-export several times during editing — they need each export preserved with a name and notes, not silently overwritten.

The `book_manuscripts` table tracks revisions; `books.word_doc_storage_path` mirrors the *active* one for legacy callers.

## Flow: list

```js
stos_title_manuscripts_list({ titleId })
```

Returns rows with `id`, `storage_path`, `display_name`, `notes`, `bytes`, `uploaded_at`.

## Flow: register a new revision

The user uploads the file themselves (the plugin doesn't upload files directly — the user goes through the dashboard's upload UI or generates a presigned URL via the in-app flow). Once uploaded:

```js
stos_title_manuscripts_create({
  titleId,
  storagePath: 'user_uuid/book_uuid/2026-05-13-editor-pass-2.docx',
  displayName: 'Editor pass 2',
  notes: 'Round of dev edits from Sarah Marsh; trimmed 4k words from act 2.',
})
```

## Flow: switch the active revision

```js
stos_title_manuscripts_set_active({ id })
```

This mirrors the storage path onto the parent title's `word_doc_storage_path`. Subsequent reads (chapter drafting, story-bible extraction) use the newly-active file.

## Flow: rename / annotate

```js
stos_title_manuscripts_update({ id, displayName: 'Final approved', notes: 'Last copy edit pass.' })
```

## Flow: delete

```js
stos_title_manuscripts_delete({ id })
```

The DB row is removed; the underlying storage file is **not** auto-deleted by this call (kept for accidental-recovery safety). Tell the user this so they can clean storage manually if needed.

## Anti-patterns

- **Auto-setting a new upload as active.** Don't — the user may want to register it for archival without switching to it yet.
- **Treating the active revision as the only one.** Each revision is independent; older ones stay accessible.
