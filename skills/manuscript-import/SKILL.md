---
name: manuscript-import
description: Use when the user wants to import, upload, or bring in a manuscript file (.docx or Markdown) so its chapters and scenes land in a StorytellerOS title. Triggers on "import my manuscript", "upload this draft to [title]", "bring my book into StorytellerOS", "parse this file into chapters", "add these chapters from my Word file". Always asks whether to only store the manuscript or also build the story bible before uploading.
---

# Manuscript Import

## What this does

`stos_titles_upload_manuscript` is the same import the app runs from **Upload Manuscript**. It splits the file into chapters, saves every chapter's full text with a first scene for each, keeps a copy of the file in the title's Manuscripts list, and adds any front or back matter (dedication, acknowledgements, about the author and so on) to the title.

The author's words are the point. Nothing in this flow rewrites, trims, or reorders them.

## Step 1: Find the title

```js
stos_pen_names_list()                         // match the pen name
stos_titles_list({ penNameId })               // match the title
stos_chapters_list({ bookId: titleId })       // does it already have chapters?
```

No title yet? Create one first: `stos_titles_create({ fields: { title: '...' }, penNameId })`. Confirm the pen name and title with the author before going on.

## Step 2: Check the file

- **Accepted:** Word (`.docx`) and Markdown (`.md` or `.markdown`), up to 50 MB.
- **Not accepted:** older Word (`.doc`), Apple Pages (`.pages`) and `.rtf`. Tell the author how to convert in plain words: in Word, **Save As → Word Document (.docx)**; in Pages, **File → Export To → Word**.
- **How chapters are found:**
  - Word: chapter titles set in the **Heading 1** or **Heading 2** style.
  - Markdown: a heading line per chapter, written `# Chapter 1: Title`. When the file has these, only those headings start chapters, and every other line stays in the chapter text, including numbered lines, dates and lines that begin with "Chapter".
  - Headings named Dedication, Epigraph, Foreword, Preface, Acknowledgements or About the Author fill those parts of the title. If the title already has one, the new text is added after it, never over it.
  - Prologue and Epilogue become their own chapters.
  - A file with no headings at all becomes one chapter.

If you are preparing a Markdown file for the author (for example from text they pasted), give every chapter a `# Chapter N: Title` heading and copy their text exactly.

- **Existing chapters decide the mode.** If `stos_chapters_list` returned chapters, explain the two choices in plain words and let the author pick:
  - `append` (the default): adds the file's chapters after the ones already there.
  - `merge`: adds only the chapter numbers the title doesn't have yet, so a file of chapters 1 to 20 on a title that has 1 to 12 adds 13 to 20.
  - Replacing the existing chapters is not available here. If the author wants that, they do it in the app.

## Step 3: Ask before uploading (required)

Before calling the upload tool, ask the author this, in these words or close to them:

> "Do you want me to just import and store the manuscript, or also build the story bible (characters, places, events and lore) from it?"

Wait for the answer. Don't upload until you have it, and don't start building the story bible unless the author says yes to it. "Just import" is a complete, good answer.

## Step 4: Upload

Read the file's bytes, base64-encode them, and call:

```js
stos_titles_upload_manuscript({
  id: titleId,
  filename: 'my-book.docx',          // must end in .docx, .md or .markdown
  content_base64: '<file bytes, base64>',
  conflict_resolution: 'append',     // or 'merge'; omit for append
})
```

Call it once per file. Never paste the manuscript into the chat to retype it.

## Step 5: Report what happened

A successful response carries:

| Field | Tell the author |
|---|---|
| `chapters_created` | How many chapters were added. With `merge` this can be lower than the number of chapters in the file; say which numbers were skipped because the title already had them. |
| `total_word_count` | How many words the file held. |
| `warnings` | Every warning, in full and in plain words. They explain things like no headings found, chapter numbers out of order, numbered titles such as "1. The Arrival" that weren't treated as chapters, or front matter added after existing text. Suggest the fix the warning names. |
| `source_file_saved` | If `false`, the chapters are in, but the copy of the file wasn't saved to the Manuscripts list. Say so. |
| `chapter_ids` | Keep these for the next steps; don't show raw ids. |

A good report: "Imported 24 chapters, about 86,000 words, into *Curses and Currents*." Then the warnings, if any.

If the response has no `warnings` or `chapter_ids` fields, the server is an older version; report chapters and words and move on.

## Step 6: Follow the author's answer from Step 3

- **Just import and store:** you're done. Mention that the story bible can be built any time later.
- **Also build the story bible:** only after a successful import, follow the [`story-bible`](../story-bible/SKILL.md) skill. It defaults to the chat loop, which reads the imported chapters and lets the author steer as it goes. It uses the server extractor only for unattended runs or very large books; that extractor reads the chapters you just imported. Don't repeat its steps here; switch to that skill.

## When something goes wrong

- **Partial import (an error after some chapters were saved).** The error text ends with `(details: {"committed_chapter_ids": [...], "warnings": [...]})`, and the message names the chapter where it stopped.
  1. Read `committed_chapter_ids`, then `stos_chapters_list({ bookId: titleId })` to confirm what actually landed.
  2. Tell the author which chapters are in and where it stopped. Their existing chapters were not touched.
  3. **Never re-upload the whole file.** That adds the saved chapters a second time, and front and back matter that were already saved get added again too.
  4. The safe retry is a file holding only the chapters that didn't make it, without the front or back matter headings, uploaded with `append`. Ask the author before retrying.
- **"The manuscript has no text to import" (422):** the file has headings but no chapter text, or is empty. Ask the author to check they picked the right file.
- **Paused (409 `STRUCTURE_OPS_PAUSED`):** nothing was written. Tell the author this kind of change is paused for now and their text is safe. Don't retry.
- **Wrong file type or too large:** use the conversion steps in Step 2, or suggest splitting the book into two files and importing the second with `append`.

## Removing a chapter after an import

Sometimes an import leaves a chapter the author doesn't want, such as a duplicate. Chapter delete is two steps, and a copy of the chapter is saved first so it can be brought back.

1. Confirm with the author which chapter you mean (number and title).
2. Call `stos_chapters_delete({ id })` **without** `confirm`. Nothing is deleted. The result is `{ confirmation_required: true, question }`: put that question to the author.
3. Only if the author says yes, call `stos_chapters_delete({ id, confirm: true })`. The response includes a `revisionId`.
4. To bring it back: `stos_chapters_restore_deleted({ titleId, revisionId })`. If its old chapter number is now taken, that call also returns a question; ask it, and call again with `confirm: true` only on a yes. Deleted chapters are also listed on the title's Revisions page in the app, with a Restore button.

A 409 `TRY_AGAIN` means nothing was deleted; call again. If the server answers that chapter delete is paused (409 `STRUCTURE_OPS_PAUSED`), or `stos_chapters_restore_deleted` isn't among your tools, chapter delete isn't available yet: nothing changed, so tell the author that and don't look for another way to remove the chapter.

## Anti-patterns

- **Uploading before asking the Step 3 question.**
- **Starting a story bible build without a yes.** Importing and building the bible are separate choices.
- **Re-uploading a whole file after a partial failure.** Read `committed_chapter_ids` first; it duplicates chapters and front matter.
- **Hiding warnings.** They tell the author what to fix in their file.
- **Calling `stos_chapters_delete` with `confirm: true` on the first call,** or deleting chapters to "clean up" without the author choosing each one.
- **Rewriting the author's text** while preparing a Markdown file. Add headings only.

## Related skills

- [`story-bible`](../story-bible/SKILL.md): build characters, places, events and lore from the imported book.
- [`manuscript-revisions`](../manuscript-revisions/SKILL.md): the saved manuscript files for a title, and which one is active.
- [`chapter-drafting`](../chapter-drafting/SKILL.md): read and revise the imported chapters.
