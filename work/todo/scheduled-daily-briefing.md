# Scheduled Daily Briefing

Create a scheduled process that runs daily, invokes Claude Code with the relevant skills, and delivers the combined output via email or notification.

## Skills to Include

| Skill | Description | Include? | Notes |
|-------|-------------|----------|-------|
| `mail-check` | USPS mail pieces and packages | ❌ Excluded | Requires interactive login |
| `weather-check` | 3-day NWS forecast for Brier, WA | ✅ Always | |
| `calendar-check` | Upcoming events for the next 7 days | ✅ Always | |
| `gmail-check` | Gmail inbox grouped by theme | ✅ Always | |
| `meal-check` | Weekly dinner plan from Google Drive | ✅ Always | |
| `canvas-inquisitor` | Grade report for all observed Canvas students | ✅ Always | |
| `hawk-highlighter` | MTHS Hawk Highlights newsletter diff | ✅ Conditional | Only runs when a new issue has arrived since last run |
| `finance-check` | Account balances across BECU and Heritage Bank NW | ❌ Excluded | Requires interactive login |
| `morning-report` | Bundle skill covering finance/mail/weather/calendar/gmail/meals | ❌ Excluded | Using individual skills instead |
| `resume-builder` | Resume tailoring and cover letter generation | ❌ Excluded | Not a daily briefing skill |

## Requirements

- Runs on a daily schedule (likely morning)
- Invokes Claude Code (CLI or API) to execute the skills
- Captures the combined output
- Delivers output to Josh (email or push notification)
- Should be low-maintenance once set up — no manual trigger needed

## Open Questions

- Where does the process run? (local machine via Task Scheduler, home server, cloud function, GitHub Actions, etc.)
- Does Claude Code CLI support non-interactive / headless execution of skills?
- Should each skill run independently and output be combined, or run as a single session?
- Delivery mechanism: email (Gmail MCP), push notification, or other?
- How does hawk-highlighter detect "new issue since last run"? Needs a state file or last-seen timestamp.
