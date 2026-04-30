# Claude Code Workspace

This is a personal monorepo workspace containing multiple repositories as git submodules under the `repos/` directory.

## Project Structure

- `repos/` - Git submodules for individual repositories
- `.claude/skills/` - Junctions to skills in `repos/rolstad-home-skills` (auto-loaded by Claude Code)
- `work/` - Task tracking (todo, doing, done)
- `solutions/` - Solutions and experiments
- `research_reports/` - Research and analysis outputs

## Skills

Skills from `repos/rolstad-home-skills` are auto-loaded via directory junctions in `.claude/skills/`. On a fresh clone, recreate them with:

```powershell
$skillsBase = "$(pwd)\repos\rolstad-home-skills\.claude\skills"
$target = "$(pwd)\.claude\skills"
New-Item -ItemType Directory -Path $target -Force | Out-Null
Get-ChildItem $skillsBase -Directory | ForEach-Object {
    New-Item -ItemType Junction -Path (Join-Path $target $_.Name) -Target $_.FullName -Force | Out-Null
}
```

Available skills: `/finance-check`, `/mail-check`, `/morning-report`, `/weather-check`, `/hawk-highlighter`, `/calendar-check`, `/canvas-inquisitor`, `/resume-builder`, `/gmail-check`, `/meal-check`

## Conventions

- Each repository in `repos/` is an independent submodule with its own language, build system, and conventions. Always check the individual repo for its specific setup before making changes.
- Do not modify submodule references without explicit instruction.
- When working on a specific repo, navigate into it and follow its own CLAUDE.md or AGENTS.md if present.

## Best Practices

### General
- Prefer editing existing files over creating new ones.
- Do not add features, abstractions, or error handling beyond what the task requires.
- Write no comments unless the reason is non-obvious (a hidden constraint, a workaround, a subtle invariant).
- Do not add security vulnerabilities (command injection, XSS, SQL injection, etc.). Fix insecure code immediately if noticed.

### Working with Submodules
- Never `git add` or commit changes inside a submodule unless explicitly asked.
- When a task involves a specific repo, read that repo's CLAUDE.md or AGENTS.md first.
- Use absolute paths when referencing files across submodules to avoid ambiguity.

### Research and Solutions
- Save research outputs to `research_reports/` with a descriptive filename and date prefix (e.g., `2026-04-29-topic.md`).
- Save reusable solutions or scripts to `solutions/` with a clear name reflecting the problem solved.

### Task Tracking
- Use `work/todo/` for planned items, `work/doing/` for in-progress items, and `work/done/` for completed items.
- Move files between folders as status changes rather than editing status inline.

## Validation

Before considering any change complete:

1. **Correctness**: Verify the code or content does exactly what was asked — no more, no less.
2. **Submodule safety**: Confirm no unintended submodule pointer changes (`git submodule status`).
3. **Style compliance**: Check that code follows the language's standard style (PEP 8, shellcheck, etc.).
4. **Edge cases**: Identify at least 2-3 failure states or implicit assumptions and handle or document them.
5. **No regressions**: If modifying an existing repo, run its test suite before reporting done.

> Follow instructions step-by-step. Do NOT skip steps or improvise alternatives. If a step fails, STOP and report the failure.

## Build and Test

No global build step. Each submodule has its own build and test process. Navigate into the relevant `repos/<name>/` directory and follow that project's instructions.

## Code Style

- Markdown: Use ATX-style headers (`#`), fenced code blocks with language tags.
- YAML: 2-space indentation.
- Scripts: Follow the language's standard style (PEP 8 for Python, shellcheck for Bash).
