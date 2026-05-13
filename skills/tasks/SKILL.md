---
name: tasks
description: Use when the user wants to review, filter, prioritize, complete, or manage existing tasks — beyond one-shot quick capture. Triggers on "what's on my task list", "show me high-priority tasks", "what's due this week", "mark [task] done", "reschedule [task]". For one-line adds, use `quick-task-capture` instead.
---

# Tasks (full management)

## When to use this vs. `quick-task-capture`

| Situation | Skill |
|---|---|
| "Remember to email the cover designer" | `quick-task-capture` |
| "Show me everything due this week" | `tasks` |
| "Mark the cover-designer task done" | `tasks` |
| "What high-priority tasks do I have for *Curses and Currents*?" | `tasks` |
| "Reschedule the editing task to Friday" | `tasks` |

## Flow: review

```js
stos_tasks_list({ status: 'to-do', penNameId, bookId: titleId })
```

Filter combinations:
- `status` = `to-do` / `in-progress` / `completed`
- `priority` = `high` / `medium` / `low`
- `bookId`, `seriesId`, `penNameId` for scope
- `item_type` if the user differentiates personal vs. project tasks

## Flow: complete

```js
stos_tasks_complete({ id })  // sugar over PATCH status=completed
```

## Flow: edit

```js
stos_tasks_update({ id, fields: { due_date: '2026-05-20', priority: 'high' } })
```

## Flow: bulk views

If the user asks "what should I focus on this week", combine:

```js
stos_tasks_list({ status: 'to-do', penNameId })  // current
stos_tasks_list({ status: 'in-progress', penNameId })  // already started
```

Sort by `priority` and `due_date` locally. Surface the top 5–10, not the entire list.

## Anti-patterns

- **Listing 200 tasks when the user wanted a focus view.** Filter and rank.
- **Auto-completing without confirmation.** Always ask before marking something done if the user phrased it ambiguously ("the cover designer thing").
- **Creating a duplicate when they meant to edit.** Search first.
