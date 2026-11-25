# /sleep-check Skill

Create a `/sleep-check` skill that reports CPAP therapy from the `myair` MCP server (`repos/myair-mcp`): last night, recent nights, averages, compliance, and nights worth a look.

---

## Plan

### Location and Files

Follows the `rolstad-home-skills` conventions (see `home-weather` and `thermostat-summary`):

```
repos/rolstad-home-skills/.claude/skills/sleep-check/
├── SKILL.md                 # Frontmatter + steps (<500 lines)
├── config.yaml              # Flag/compliance thresholds, edit here
├── scripts/
│   ├── summarize.py         # Stdlib-only CLI: records JSON in, summary JSON out
│   └── test_summarize.py    # pytest
└── assets/
    └── report-template.md   # Output structure, loaded at render time
```

The `myair` MCP server stays unchanged. All summary math lives in the skill.

**`config.yaml`** holds every threshold (flat `key: number` pairs), so they can be changed without editing code: `flag_min_usage_hours: 4`, `flag_max_ahi: 5`, `flag_max_leak_l_per_min: 24`, `compliance_min_hours: 4`, `compliance_target_pct: 70`. The script reads it with a small strict parser that keeps it stdlib-only. Unknown keys, missing keys or non-numeric values are errors, and nothing silently falls back to defaults. `--config PATH` overrides the location for tests.

### Frontmatter

- **name:** `sleep-check`
- **description:** Reports CPAP therapy data from ResMed myAir: last night's usage, AHI, mask leak and myAir score, a table of recent nights, averages, and the 4-hour compliance rate. Defaults to the last 7 nights and accepts a number of nights. Use when the user asks "sleep check", "how did I sleep", "CPAP report", "myAir score", "how's my CPAP", "AHI last week", or "sleep the last N nights". Do not use for sleep tracking from other devices, for medical advice or interpreting results beyond the plain thresholds, or for changing device settings. Requires the myair MCP server.

### Data Sizes (measured 2026-11-24)

| Window | Records | Payload |
|--------|---------|---------|
| 7 nights | 7 | 1.4K chars |
| 30 nights | 30 | 6K chars |
| 90 nights | 90 | 18K chars |
| All (365+) | 266 | 53K chars |

Everything comes back inline with no spillover files, unlike `home-weather`.

### Computation: `scripts/summarize.py`

The averages and compliance percentages must be exact, so the script computes them instead of the model. It uses only the standard library, so any Python 3.10+ works, like the other skills' scripts.

**Getting the data in:** these responses come back inline (see Data Sizes), so there is no spillover file to point the script at. The model pipes the exact JSON array returned by `get_sleep_records` into the script on stdin, using a quoted heredoc so the shell doesn't touch it:

```bash
python <skill-dir>/scripts/summarize.py --days 7 <<'JSON'
[ ...exact JSON array from get_sleep_records... ]
JSON
```

The script also accepts a file path, for when a large "all time" response spills over to a `tool-results` file. Copying the records is cheap at 7–90 nights (1.4K–18K chars). At all time (53K chars), the model should use the spillover file if there is one.

**CLI:** `summarize.py [--days N] [--today YYYY-MM-DD] [path]`. `--today` defaults to the local date and is there so tests are deterministic.

**Output JSON:**

| Key | Contents |
|-----|----------|
| `window` | `start`, `end`, `nights` (N). Dates with no record count as unused nights |
| `latest` | Newest record, plus `usage_hm` (e.g. `7h 05m`) and `days_ago` |
| `nights` | Per-night rows (newest first) with `usage_hm` and per-night `flags`, when N ≤ 14 |
| `weeks` | Weekly rollups (weeks starting Monday: nights used, average usage, AHI, score), when N > 14 |
| `summary` | `nights_used`, `avg_usage_hm`, `avg_ahi`, `avg_leak`, `avg_score` (over used nights only), `compliance_4h_pct` (over all N nights) |
| `flags` | Nights with usage under 4h, AHI over 5, or leak over 24 L/min |
| `trend` | First-half vs. second-half deltas for usage and AHI, or `null` with fewer than 4 used nights |

Thresholds come from `config.yaml` and are echoed in the output under `thresholds`, so the report states the values it used.

**Input format (verified):** FastMCP returns a list as **one text block per record**, so the model sees a run of separate pretty-printed JSON objects, not an array. The script accepts that, a JSON array, or a Claude Code tool-results file (a list of `{"type": "text", "text": ...}` blocks).

**Validation:** input that is invalid JSON or not a list exits non-zero with a one-line message. Records without a `date` are skipped and counted in a `skipped` field.

**Tests:** `scripts/test_summarize.py` (pytest) covers the window and missing-date counting, zero-usage nights left out of the averages but counted in compliance, the switch from nights to weeks at 15+ nights, each flag, the trend with too few nights, empty input, and bad JSON.

### Steps (SKILL.md)

1. **Window:** default to **7** nights. Parse "last N nights/days", "past month" (30), "past 3 months" (90), "all time" (3650). Minimum 1.
2. **Freshness:** call `mcp__myair__get_devices()` and read `last_sleep_data_report_time`. If it is before ~4am local today, last night has not uploaded yet, so add a note to the report.
3. **Data:** call `mcp__myair__get_sleep_records(days: N)`.
4. **Summarize:** pipe the returned array into `scripts/summarize.py --days N`, or pass the spillover path. Report only the script's numbers and don't recompute anything.
5. **Render:** load `assets/report-template.md` and fill it in from the script output plus the freshness note.
6. **Follow-up offer:** past 30 nights, compare with the previous period, or drill into one night.

### Report Contents

- **Header:** "Sleep Check: last N nights (date range)", plus a device line (AirSense 11 AutoSet, last upload time in local time).
- **Last night:** usage as `7h 05m`, AHI, leak in L/min, mask on/off count, and the myAir score with its four sub-scores (`usage 70/70 · AHI 5/5 · mask 5/5 · leak 20/20`).
- **Nights table:** shows every night up to 14. Above 14 nights, it shows **weekly rollups** (week of, nights used, average usage, average AHI, average score) so the table stays readable. Unused nights show as "not used".
- **Summary:** nights used (X of N), average usage, AHI and leak over used nights, average myAir score, and **compliance** (the percentage of nights with ≥4h of use, the standard insurance measure; 70% is the usual bar).
- **Flags:** nights with usage under 4h, AHI over 5, or 95th-percentile leak over 24 L/min (ResMed's large-leak level). They are listed plainly, with no interpretation.
- **Trend line:** one sentence comparing the first half of the window with the second half (usage and AHI). Skipped when there are fewer than 4 used nights.

### Not Included

- Not added to `/morning-report`, because health data should not end up in a shared daily report by default.
- No medical interpretation beyond the fixed thresholds above. The report footer points to the care provider for questions.

### Edge Cases

- **Last night not uploaded yet:** report the latest available night and label it by date. Never present an older night as "last night".
- **Zero-usage nights:** count them in `nights_in_window` and compliance, and leave them out of the averages.
- **No used nights in the window:** show the device and freshness lines and say there is no therapy data for the window.
- **Script error (bad JSON):** show the script's message and don't fall back to computing by hand.
- **myair server not loaded:** the tool call fails. Tell the user to check the `myair` entry in `~/.claude.json` and restart Claude Code.
- **Auth error from the server:** report it once and do not retry, because repeated logins can lock the account.

### Implementation Sequence

1. Write `scripts/summarize.py` and `scripts/test_summarize.py`, run the tests, then run it once against real records.
2. Write `SKILL.md` and `assets/report-template.md`.
3. Update the `rolstad-home-skills` README and CLAUDE.md skill table, and add `myair` to "MCP Servers Required".
4. Create the junction `.claude/skills/sleep-check` in the workspace and add `/sleep-check` to the workspace `CLAUDE.md` skill list.
5. After restarting Claude Code, test with `/sleep-check`, `/sleep-check 30` and `/sleep-check all time`, and check the numbers against the myAir app for a couple of nights.
6. Commit and push `rolstad-home-skills`, then bump its submodule pointer in the workspace.
7. Move this file and `myair-mcp.md` to `work/done/`.

### Addendum (2026-11-24): Summary-First Overview

The report now opens with a summary in the style of canvas-inquisitor: 2–3 plain-English sentences and a table comparing the last 7 nights against the last 30, with 🟢/🔴/⚪ trend dots. `summarize.py` emits an `overview` block (`recent`, `baseline`, `comparison`), and the steady bands are the `trend_*` keys in `config.yaml`. The skill always fetches at least 30 nights. The Window Totals section is omitted when the window is 7 or 30 nights.
