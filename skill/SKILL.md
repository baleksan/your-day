---
name: day_time_manager
description: >
  Plan your day by entering 3-4 things you want to accomplish, each with a
  size (S = 30 min, M = 1 hr, L = 2 hr). The skill reads your Google Calendar
  for today, finds free slots, and creates a "focus time" meeting with yourself
  for each task — so your calendar reflects your real priorities.
  Trigger phrases: "plan my day", "schedule my tasks", "block time for",
  "day time manager", "time block", "focus time".
---

# Day Time Manager

> Block time on your calendar for the things that actually matter today.

## How it works

1. **Collect tasks** — ask the user for 3-4 items, each with a size.
2. **Read today's calendar** — find all existing events.
3. **Find free slots** — compute gaps between events (within work hours).
4. **Schedule focus blocks** — create a calendar event for each task in the
   best available slot.

## Size definitions

| Size | Duration | Typical use |
|------|----------|-------------|
| **S** | 30 min | Quick review, email follow-up, small PR |
| **M** | 1 hour | Feature work, design doc section, code review |
| **L** | 2 hours | Deep work, architecture, complex debugging |

## Step 1 — Gather tasks

Use `AskUserQuestion` to collect tasks from the user. Ask for up to 4 tasks.
Each task needs:
- A short description (what to accomplish)
- A size: **S** (30 min), **M** (1 hr), or **L** (2 hr)

Present it as a simple prompt. Example interaction:

```
What do you want to accomplish today? Give me up to 4 tasks with sizes.

Format: <task description> - <S|M|L>

Example:
  1. Review PR for auth service - S
  2. Write design doc for caching layer - L
  3. Fix flaky test in CI - M
  4. 1:1 prep for skip-level - S
```

If the user gives free-text, parse it — be flexible with formatting.
Accept any of: `S/M/L`, `small/medium/large`, `30m/1h/2h`.

## Step 2 — Read today's calendar

Use `mcp__plugin_google_google__calendar_events` to fetch today's events:
- `start_date`: today (YYYY-MM-DD)
- `end_date`: today (YYYY-MM-DD)
- `calendars`: `"all"` — so we see conflicts across all calendars

Parse the events to build a list of busy blocks: `[(start_time, end_time, title), ...]`

## Step 3 — Find free slots

**Work hours**: default 9:00 AM – 6:00 PM (user's local time). Respect
existing events — treat them as immovable.

**Algorithm** (run via `mcp__plugin_aisuite_aisuite__python`):

1. Parse all existing events into `(start, end)` intervals.
2. Merge overlapping intervals.
3. Compute gaps within work hours.
4. Sort tasks by size **descending** (L first, then M, then S) — large
   blocks are hardest to place, so schedule them first.
5. For each task, find the **first gap** that fits its duration.
6. Place the task at the **start** of that gap (pack early, leave
   flexibility later in the day).
7. If a task doesn't fit anywhere, flag it and tell the user.

**Constraints**:
- Never double-book — respect ALL existing events.
- Leave a 5-minute buffer between the end of an existing event and the
  start of a new focus block (bio break).
- Prefer morning slots for L tasks (deep work) when possible.
- If two gaps are equally good, pick the earlier one.

## Step 4 — Present the plan & confirm

Before creating any events, show the user the proposed schedule:

```
Here's your focus-time plan for today:

  09:00 – 11:00  [L]  Write design doc for caching layer
  11:05 – 12:05  [M]  Fix flaky test in CI
  13:00 – 13:30  [S]  Review PR for auth service
  14:00 – 14:30  [S]  1:1 prep for skip-level

  Existing events preserved:
  12:00 – 13:00  Team standup
  15:00 – 16:00  Sprint planning

Shall I create these calendar events?
```

Use `AskUserQuestion` with options:
- "Create all events" (recommended)
- "Let me adjust first"

If the user wants adjustments, re-collect their preferences and re-run
the slot finder.

## Step 5 — Create calendar events

For each scheduled task, use `mcp__plugin_aisuite_aisuite__python` with
`mcp.call()` to create calendar events via the Google Calendar API:

```python
result = mcp.call("plugin:google:google:calendar_create_event",
    title="[Focus] <task description>",
    start="<ISO datetime>",
    end="<ISO datetime>",
    description="Time-blocked by Day Time Manager\nSize: <S|M|L>\nGoal: <task description>"
)
```

**Event formatting**:
- Title prefix: `[Focus]` — so it's visually distinct on the calendar
- Description: include the size and goal
- No attendees (meeting with yourself)
- Default calendar (primary)

If `calendar_create_event` is not available as an MCP tool, fall back to
generating Google Calendar quick-add links that the user can click to
create each event manually. Build URLs of the form:
```
https://calendar.google.com/calendar/render?action=TEMPLATE&text=[Focus]+<title>&dates=<start>/<end>&details=<description>
```

## Step 6 — Summary

After creating events, show a clean summary:

```
Done! Your day is planned:

  09:00 – 11:00  [L]  Write design doc for caching layer
  11:05 – 12:05  [M]  Fix flaky test in CI
  13:00 – 13:30  [S]  Review PR for auth service
  14:00 – 14:30  [S]  1:1 prep for skip-level

  Total focus time: 4 hours scheduled
  Free time remaining: 2 hours (after existing meetings)

Have a productive day!
```

## Error handling

- **No free slots**: "Your calendar is packed today. Consider moving or
  declining a meeting to make room, or pick fewer/smaller tasks."
- **Partial fit**: Schedule what fits, list what didn't, suggest tomorrow.
- **Calendar API error**: Show the error, offer the manual link fallback.
- **No tasks provided**: Re-prompt. Minimum 1 task, maximum 4.

## What NOT to do

- Do not move or delete existing events. Ever.
- Do not schedule outside work hours unless the user explicitly asks.
- Do not create events without showing the plan and getting confirmation.
- Do not over-optimize — simple first-fit-descending is good enough.
