---
name: timeline
description: Use when the user wants to manage the IN-UNIVERSE story timeline — when things happen inside the fictional world. Triggers on "add a story event", "when does the war start in [title]", "what's the canon timeline", "list events in [series]". Distinct from `calendar` (real-world dates).
---

# Story Timeline

## What this is

The `events` table in STOS holds in-universe events: "The king dies", "Sasha arrives in the cove", "the war begins." Each has a free-text `event_date` (because in-universe dates often don't map to real calendars), an `act`, an `importance` rank, and an `event_order` for sorting within a series.

Distinct from `calendar` events (real-world dates).

## Flow: add a story event

```js
stos_events_create({
  fields: {
    event_name: 'The Crow Court convenes for the first time in 400 years',
    event_date: 'Year 1187, Hightide Moon',
    event_type: 'political',
    description: '...',
    act: 'act-two',
    importance: 'major',
    event_order: 14,
    development_notes: 'Triggers Sasha\'s arc reversal in chapter 17.',
  },
})
```

Then wire it where it lands:

```js
stos_worldbuilding_link({ entityType: 'scene', entityId: sceneId, targetType: 'event', targetId: eventId })
stos_worldbuilding_link({ entityType: 'event', entityId: eventId, targetType: 'character', targetId: characterId })
```

## Flow: review the timeline

```js
stos_events_list({ penNameId })
```

Sort by `event_order` or by `act` to show the in-universe sequence.

## Anti-patterns

- **Using `calendar-events` for in-universe dates.** Wrong table. The calendar is for real-world dates only.
- **Skipping the associations.** A timeline event that isn't linked to a scene or character is hard to find later — wire it on creation.
