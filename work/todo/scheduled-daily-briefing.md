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
| `hawk-highlighter` | MTHS Hawk Highlights newsletter diff | ✅ Always (MVP) | Conditional detection deferred — requires state file + git push |
| `finance-check` | Account balances across BECU and Heritage Bank NW | ❌ Excluded | Requires interactive login |
| `morning-report` | Bundle skill covering finance/mail/weather/calendar/gmail/meals | ❌ Excluded | Using individual skills instead |
| `resume-builder` | Resume tailoring and cover letter generation | ❌ Excluded | Not a daily briefing skill |

## Requirements

- Runs as a **Claude routine** (remote scheduled agent via `/schedule`) so it runs autonomously in the cloud with no dependency on a local machine being on or logged in
- Runs daily at **6:00 AM Pacific** (cron `0 13 * * *` UTC in PDT; update to `0 14 * * *` in November for PST)
- Delivers a single combined HTML email to **jrolstad@gmail.com**
- Should be low-maintenance once set up — no manual trigger needed

## Implementation Plan

### Prerequisites (manual steps before building)

1. Connect **google-workspace** MCP connector at https://claude.ai/customize/connectors
2. Connect **canvas-mtgibbs** MCP connector at https://claude.ai/customize/connectors
3. Connect **GitHub** — run `/web-setup` or install the Claude GitHub App so the routine can clone the repo

### Step 1 — Create `daily-briefing` skill in rolstad-home-skills

Create two files in `repos/rolstad-home-skills/.claude/skills/daily-briefing/`:

**`config.yaml`**
```yaml
sender_email: jrolstad@gmail.com
recipient: jrolstad@gmail.com
```

**`SKILL.md`** — headless orchestrator that inlines the fetch + report steps from each child skill (weather, calendar, gmail, meals, canvas, hawk-highlights), explicitly skips all interactive follow-up prompts, then assembles and sends a single combined HTML email.

Skill steps:
1. Load `config.yaml` for sender/recipient
2. **Weather** — run weather-check steps 1–2, skip follow-up prompt
3. **Calendar** — run calendar-check steps 1–2, skip follow-up prompt
4. **Gmail** — run gmail-check steps 1–2, skip follow-up prompt
5. **Meals** — run meal-check steps 1–3 (includes `python .claude/skills/meal-check/extract_docx_text.py`), skip follow-up prompt
6. **Canvas** — run canvas-inquisitor steps 0–3, skip follow-up prompt
7. **Hawk Highlights** — run hawk-highlighter steps 0–3, skip follow-up prompt
8. Assemble all six sections as HTML `<h2>` blocks and send via:
   ```
   mcp__google-workspace__send_gmail_message(
     to: recipient,
     subject: "Daily Briefing — [Weekday, Month Day]",
     body: [combined HTML with inline styles],
     body_format: "html",
     user_google_email: sender_email
   )
   ```

> **Why inline steps instead of calling sub-skills:** Skill invocation (`/skill-name`) is a user-facing interactive mechanism and does not work in a headless routine. The `daily-briefing` SKILL.md must inline the data-fetch and report-generation steps from each child skill directly.

Also add the `daily-briefing` directory junction to `.claude/skills/` in the workspace (consistent with the other skills).

### Step 2 — Create the Claude routine (via `/schedule`)

- **Source repo:** `https://github.com/jrolstad/rolstad-home-skills` (use the skills repo directly, not the workspace wrapper — so relative paths like `python .claude/skills/meal-check/extract_docx_text.py` resolve correctly in the remote checkout)
- **Cron:** `0 13 * * *` (6:00 AM PDT)
- **MCP connectors:** google-workspace + canvas-mtgibbs
- **Model:** claude-sonnet-4-6
- **Prompt:**
  ```
  You are running the daily home briefing for Josh Rolstad.

  Read the full instructions in .claude/skills/daily-briefing/SKILL.md and execute them exactly.

  Important:
  - This is an automated, unattended run — do NOT pause for user input or offer follow-up actions.
  - Run every skill section fully before moving to the next.
  - Send the combined email at the end as instructed in the SKILL.md.
  - If a section fails (e.g., Canvas is unavailable), note the failure in the email and continue with the remaining sections.
  ```

### Step 3 — Verify

1. Trigger the routine immediately via `RemoteTrigger {action: "run"}`
2. Monitor status at https://claude.ai/code/routines
3. Confirm the combined briefing email arrives at jrolstad@gmail.com with all six sections

## Future Enhancements

- **Hawk-highlighter conditional detection:** Store last-seen newsletter message-id in a state file in the repo. Routine reads state, checks newest Gmail message-id, only runs diff if different, commits updated state back via GitHub.
- **PST/PDT cron shift:** Update cron in November (`0 14 * * *`) and March (`0 13 * * *`) to keep 6:00 AM Pacific year-round.
- **Failure alerting:** If the routine itself fails entirely, send a fallback notification.
