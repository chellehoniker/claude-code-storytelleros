---
description: Quick-capture a task into StorytellerOS
argument-hint: <what to remember>
allowed-tools: [Bash, Read, Write]
---

# Quick Task

Capture this as a task in StorytellerOS: $ARGUMENTS

Follow the quick-task-capture skill: extract a short title, optional description, and call `stos_tasks_create({ fields: { task_name, description } })`. Confirm the task ID back to the user.
