---
name: time-tracking
description: Use when the user wants to start, stop, or review time spent writing. Triggers on "start a writing timer", "stop the timer", "how long did I spend on [project]", "log my pomodoro", "log a writing session", "my writing time this week".
---

# Time Tracking

## Three related concepts

| Concept | What it is | Tools |
|---|---|---|
| **Time-tracking entry** | A timed work block (start + stop) | `stos_time_tracking_*` |
| **Writing session** | A focused-work record (often pomodoro-style) | `stos_writing_sessions_*` |
| **Writing goal** | A target (words per day, hours per week, deadline) | `stos_writing_goals_*` |

## Flow: start / stop a timer

```js
stos_time_tracking_start({ penNameId, bookId: titleId, chapterId, note: 'Drafting' })
// later
stos_time_tracking_stop({ id })  // sets ended_at = now
```

If the user doesn't specify a pen name + title, ask which they're tracking against (or default to their active one).

## Flow: log a completed session

If the user worked offline and wants to log retroactively, use `stos_time_tracking_create` directly with both `started_at` and `ended_at` set.

## Flow: review

For "how much time did I spend on [X]":

```js
stos_time_tracking_list({ penNameId, bookId: titleId })
```

Sum the (`ended_at` − `started_at`) deltas and report. If the user wants a per-day breakdown, group locally.

## Pomodoro settings

If the user references pomodoro defaults:

```js
stos_pomodoro_settings_get()
stos_pomodoro_settings_update({ fields: { focus_minutes: 25, ... } })
```

## Writing goals

Goal-tracking is a separate flow:

```js
stos_writing_goals_list()
stos_writing_goals_create({ fields: { ... } })
```

Use this when the user says "set a goal of 1500 words/day" or "I'm aiming to finish by [date]".

## Anti-patterns

- **Auto-stopping the user's timer.** Don't stop a timer unless the user asked. They may have walked away and want the elapsed time counted.
- **Starting a timer without scope.** If you can't infer pen name + title, ask — otherwise the entry hangs unattributed.
