---
description: Start or stop a time-tracking session
argument-hint: <start [project] | stop | status>
allowed-tools: [Bash, Read, Write]
---

# Time Tracker

Time-tracking action: $ARGUMENTS

Follow the `time-tracking` skill.

- **start:** look up the title the user named (`stos_titles_list`) and call `stos_time_tracking_start({ penNameId, bookId, note })`, putting any free-text label in `note`.
- **stop:** find the running session (below), call `stos_time_tracking_stop({ id })`, and report the total elapsed time.
- **status:** call `stos_time_tracking_list` and find the session with `end_time` null. That's the one running. If there is none, say no timer is running.
