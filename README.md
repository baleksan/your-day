# Your Day

> Block time on your calendar for the things that actually matter today.

An AI Expert Suite skill that takes 3-4 things you want to accomplish, finds free slots in your Google Calendar, and creates focus-time events so your calendar reflects your real priorities.

## How it works

1. Tell the skill what you want to accomplish (up to 4 tasks)
2. Tag each task as **S** (30 min), **M** (1 hr), or **L** (2 hr)
3. The skill reads your calendar, finds gaps, and proposes a plan
4. You confirm, and it creates `[Focus]` events on your calendar

```
What do you want to accomplish today?

  1. Review PR for auth service - S
  2. Write design doc for caching layer - L
  3. Fix flaky test in CI - M
  4. 1:1 prep for skip-level - S
```

The scheduler places large tasks first (they're hardest to fit), packs early to leave flexibility later, and never double-books.

## Installation

### AI Expert Suite

Copy or symlink the skill into your Claude skills directory:

```bash
ln -s /path/to/your-day/skill ~/.claude/skills/day-time-manager
```

Then invoke it with `/day_time_manager` or just say "plan my day".

### Prerequisites

- **Google Calendar** connected via AI Expert Suite setup
- **AI Expert Suite** with the Google plugin enabled

## Size guide

| Size | Duration | Examples |
|------|----------|---------|
| **S** | 30 min | Quick PR review, email follow-up, prep notes |
| **M** | 1 hour | Feature work, design doc section, code review |
| **L** | 2 hours | Deep work, architecture, complex debugging |

## Scheduling rules

- Respects all existing events (never moves or deletes anything)
- 5-minute buffer between events (bio breaks)
- Work hours: 9 AM - 6 PM (default)
- Large tasks scheduled first (first-fit-descending)
- Always shows the plan and asks for confirmation before creating events
- Falls back to Google Calendar quick-add links if the create API isn't available
