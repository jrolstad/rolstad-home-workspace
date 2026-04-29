# rolstad-home-workspace

A personal monorepo workspace for home and hobby projects. It acts as a single entry point for all personal repositories, organized as git submodules, along with workspace-level tooling, task tracking, and research outputs.

## Structure

```
rolstad-home-workspace/
├── repos/               # Git submodules — one per project
├── work/
│   ├── todo/            # Planned tasks
│   ├── doing/           # In-progress tasks
│   └── done/            # Completed tasks
├── solutions/           # Reusable scripts and one-off solutions
└── research_reports/    # Research and analysis writeups
```

## Getting Started

Clone the workspace including all submodules:

```bash
git clone --recurse-submodules https://github.com/jrolstad/rolstad-home-workspace.git
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

Each repository under `repos/` is independent with its own language, build system, and setup instructions. See the individual repo's README for details.

## AI Tooling

This workspace is configured for use with [Claude Code](https://claude.ai/code). See `CLAUDE.md` for conventions and best practices used when working in this repo with AI assistance.
