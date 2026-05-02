---
name: quick-task-capture
description: Use when the user says "remember to X", "I need to Y later", "add a todo", "capture this", "log a task", or otherwise wants something tracked without leaving the conversation. Also the back-end for the /stos-task slash command.
---

# Quick Task Capture

## When to Use

The user wants a one-shot task captured into StorytellerOS. Triggers:

- "Remember to email the cover designer about the rebrand"
- "Add a todo: finish chapter 12 by Friday"
- "I need to log an expense for the editor later"
- The `/stos-task` slash command

## Flow

1. Extract a short, action-oriented title (≤ 80 characters). Strip filler like "remember to" or "I need to".
2. If the user gave more context than fits in the title, put it in a short description.
3. If the user named a pen name, scope it (see `pen-names` skill). Otherwise omit and let it land on the active pen name.
4. Call `stos_tasks_create({ title, description?, penNameId? })`.
5. Confirm back to the user with the task title and (if returned) the id, in one line. Don't recap the full body.

## Examples

**User:** "Remember to schedule a call with the audiobook narrator next week."

**You:** Call `stos_tasks_create({ title: "Schedule call with audiobook narrator", description: "Aim for next week" })`. Reply: "Captured: 'Schedule call with audiobook narrator'."

**User:** "/stos-task finish chapter 12 by Friday"

**You:** Call `stos_tasks_create({ title: "Finish chapter 12 by Friday" })`. Reply: "Captured: 'Finish chapter 12 by Friday'."

## Don't

- Don't ask follow-up questions for a quick capture. The whole point is speed. If something is genuinely ambiguous (e.g., "remember to do that thing"), ask once, briefly.
- Don't pull the full task list afterward. The user wanted to add one thing, not review everything.
- Don't try to schedule, prioritize, or assign — that's a different flow.
