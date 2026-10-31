---
name: manuscript-revisions
description: Use when the user wants to list, upload, or switch between manuscript file revisions for a title. Triggers on "show me my manuscript versions", "I just uploaded a new draft", "switch to the [name] revision", "what manuscripts do I have for [title]".
---

# Manuscript Revisions

## What this is

Each title can carry several saved manuscript files (revisions). Authors re-export from their writing app several times during editing, and each export should be kept with a name and notes, not silently overwritten. One revision is marked *active*.

## Flow: list

```js
stos_title_manuscripts_list({ titleId })
```

Returns rows with `id`, `storage_path`, `display_name`, `notes`, `bytes`, `uploaded_at`.

## Flow: add a new revision

To bring a manuscript file in, use the **`manuscript-import`** skill. `stos_titles_upload_manuscript` saves the file to this list *and* turns it into chapters and scenes; it asks the author first whether to also build the story bible.

`stos_title_manuscripts_create` only registers a file that is already stored in StorytellerOS (for example one the author uploaded in the app). It needs the file's stored path, which you can only get from an existing revision or the app; you can't upload a file through it:

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

This makes that file the title's current manuscript. Switching revisions does not change the chapters and scenes already in the title; those only change through an import or edits.

## Flow: rename / annotate

```js
stos_title_manuscripts_update({ id, displayName: 'Final approved', notes: 'Last copy edit pass.' })
```

## Flow: delete

```js
stos_title_manuscripts_delete({ id })
```

The revision is removed from the list, but the stored file itself is kept, so nothing is lost by accident. Tell the user this.

## Anti-patterns

- **Auto-setting a new upload as active.** Don't. The user may want to keep it for reference without switching to it yet.
- **Treating the active revision as the only one.** Each revision is independent; older ones stay accessible.
