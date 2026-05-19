---
description: Start or stop a time-tracking session
argument-hint: <start [project] | stop | status>
allowed-tools: [Bash, Read, Write]
---

# Time Tracker

Time-tracking action: $ARGUMENTS

If "start", call `stos_time_tracking_start` with the project / book label the user gave. If "stop", call `stos_time_tracking_stop` and report total elapsed. If "status", call `stos_time_tracking_list` and find the session with `ended_at` null — that's the currently running one. If none, report no active session.
