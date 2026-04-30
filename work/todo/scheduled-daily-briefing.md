# Scheduled Daily Briefing

Create a scheduled process that runs daily, invokes Claude Code with the relevant skills, and delivers the combined output via email or notification.

## Skills to Include

The following skills exist in `rolstad-home-skills` and are candidates for the daily briefing:

| Skill | Description | Include? |
|-------|-------------|----------|
| `canvas-inquisitor` | Grade report for all observed Canvas students | ✅ Include |
| `mail-check` | USPS mail pieces and packages | ✅ Include |
| `weather-check` | 3-day NWS forecast for Brier, WA | ✅ Include |

**Recommendation:** Run `mail-check` + `weather-check` + `canvas-inquisitor` daily. conditionally if a new newsletter has arrived.

## Requirements

- Runs on a daily schedule (likely morning)
- Invokes Claude Code (CLI or API) to execute the skills
- Captures the combined output
- Delivers output to Josh (email or push notification)
- Should be low-maintenance once set up — no manual trigger needed

## Open Questions

- Where does the process run? (local machine via Task Scheduler, home server, cloud function, GitHub Actions, etc.)
- Does Claude Code CLI support non-interactive / headless execution of skills?
- Should each skill run independently and output be combined, or run as a single morning-report-style session?
- Delivery mechanism: email (Gmail MCP), push notification, or other?
- `morning-report` already covers Gmail, calendar, mail, weather, and meals — should the daily run just be `morning-report` + `canvas-inquisitor`?
- Does `hawk-highlighter` run conditionally (only when a new issue exists) or always?
