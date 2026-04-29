# Resume Builder Skill

## Request

Create a new Claude Code skill called `resume-builder` in the `rolstad-home-skills` repo. Requirements:

- Given a job posting or description, tailor an existing resume to fit that position as strongly as possible
- Use resumes stored in Google Drive folder: https://drive.google.com/drive/folders/0B4BtZJ4ZrcRca05TbWxOWWd1SU0?resourcekey=0-VpymoZwrUrEVVsR9sUE46g
- Reason through what makes the resume strong for the specific position
- Ensure ATS (Applicant Tracking System) compatibility
- Be interactive — loop with the user on changes and next steps
- Validate existing Drive resumes against a job description on request
- Generate a targeted cover letter when ready, prompting for why the job is important and tailoring to the company and role

## Outcome

Created `.claude/skills/resume-builder/SKILL.md` in `rolstad-home-skills` implementing three modes:

**Build mode** — Default. Accepts a job posting (pasted text or URL), fetches the base resume from Google Drive, runs a pre-flight analysis (ATS keyword gaps, skill coverage scorecard, structural risks), generates a fully tailored ATS-safe resume, then enters an interactive revision loop. Resume can be saved back to Drive as a new Google Doc.

**Validate mode** — Lists all resumes in the Drive folder, scores selected resume(s) against the job description with a 5-category scorecard (keyword coverage, required/preferred skills, responsibility alignment, ATS structure), and recommends the strongest base version.

**Cover Letter mode** — Prompts the user with 4 targeted questions (role appeal, company admiration, anchor achievement, tone/constraints), then generates a ≤400-word cover letter that complements rather than duplicates the resume. Saved to Drive on request.

**ATS rules enforced:** plain text structure, no tables/columns/images, standard section headers, role title mirrored in summary, action-verb bullets, exact keyword phrases used naturally, .docx format recommended.

## Files Changed

- `repos/rolstad-home-skills/.claude/skills/resume-builder/SKILL.md` — new skill (created)
- `repos/rolstad-home-skills/README.md` — added `/resume-builder` to skills table
- `repos/rolstad-home-skills/CLAUDE.md` — added skill row and google-workspace MCP entry

## Completed

2026-04-29 — pushed to `rolstad-home-skills` main and submodule pointer updated in `rolstad-home-workspace`.
