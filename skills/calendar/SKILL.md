---
name: calendar
description: Use when the user wants to add, find, or manage real-world calendar events — release dates, signings, deadlines, podcast recordings. Triggers on "add to my calendar", "what's on my calendar [date]", "schedule a [thing]", "block out [date] for [thing]". DOES NOT cover story-timeline events — see `timeline` for those.
---

# Calendar (user calendar)

## What this covers vs. `timeline`

| User said | Skill |
|---|---|
| "Add [event] to my calendar on [date]" | `calendar` |
| "Block out next Friday for the launch" | `calendar` |
| "When is the audiobook narration session?" | `calendar` |
| "Add a story event for when the king dies" | `timeline` (different table) |
| "What happens between chapter 5 and 6 on the in-universe timeline?" | `timeline` |

Real-world dates → `calendar` (calendars + calendar-events tables).
In-universe story dates → `timeline` (events table, the worldbuilding one).

## Flow: add an event

1. List calendars: `stos_calendars_list`. If the user has multiple, ask which one. If only one, use it.
2. Create:

```js
stos_calendar_events_create({
  fields: {
    calendar_id,
    title: 'Audiobook narration — Curses and Currents',
    start_at: '2026-05-20T10:00:00-05:00',
    end_at: '2026-05-20T13:00:00-05:00',
    location: 'Studio Address',
    description: 'Three-hour session with Diana Logan',
  },
})
```

## Flow: review

For "what's on my calendar this week":

```js
stos_calendar_events_list({ from: '2026-05-13', to: '2026-05-20' })
```

## Anti-patterns

- **Confusing calendar with timeline.** If the user said "story event" or referenced an in-universe date, use the `timeline` skill instead.
- **Creating an event without confirming the calendar.** If the user has multiple calendars (work / personal / launches), wrong calendar = lost event.
