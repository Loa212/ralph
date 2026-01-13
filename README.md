# Ralph - Autonomous AI Development Loop

Ralph is an autonomous development loop system that uses the Codex CLI to iteratively implement features from a PRD (Product Requirements Document).

## Installation

This repository is designed to be **reused across projects** as tooling (prompts, scripts, loops), not copied around.

### Recommended: install as a Git submodule

Using a submodule lets you:

* reuse the same Ralph tooling across multiple projects
* keep a clean separation between project code and automation
* update the tooling in one place when it evolves

From the root of your project:

```bash
git submodule add \
  -b codex/update-to-use-codex-cli-instead-of-claude \
  https://github.com/Loa212/ralph \
  .ralph
```

Then commit:

```bash
git add .ralph .gitmodules
git commit -m "chore: add ralph tooling as submodule"
```

This will place the Ralph tooling under `.ralph/` while keeping its own Git history intact.

To update the tooling later:

```bash
cd .ralph
git pull
cd ..
git add .ralph
git commit -m "chore: update ralph tooling"
```

---

### Alternative: copy the files directly (not recommended)

You may also copy the contents of this repository directly into your project if you prefer not to use submodules.

However, this approach:

* removes the update path
* makes it harder to keep multiple projects in sync
* is discouraged unless you intentionally want to fork/customize the tooling per project

---

### Notes

* The `.ralph/` directory is tooling only — it does not affect runtime code.
* Specs (`specs/`) and plans (`IMPLEMENTATION_PLAN.md`) live in the host project.
* Git submodules are **recommended**, but not required.

---


## Quick Start

```bash
# 1. Create a new project
./ralph/new.sh my-feature

# 2. Edit your PRD
code ralph/projects/my-feature/prd.md

# 3. Convert PRD to JSON tasks
./ralph/convert.sh my-feature

# 4. Start the loop (with tmux monitoring)
./ralph/start.sh my-feature --monitor
```

## Prerequisites

```bash
# Install dependencies
brew install jq tmux
npm install -g @openai/codex-cli
```

## Commands

### `./ralph/new.sh <project-name>`
Create a new project from template.

```bash
./ralph/new.sh signals
# Creates: ralph/projects/signals/
```

### `./ralph/convert.sh <project-name>`
Convert your PRD.md to actionable JSON tasks using Codex.

```bash
./ralph/convert.sh signals
# Reads: ralph/projects/signals/prd.md
# Creates: ralph/projects/signals/prd.json
```

### `./ralph/start.sh <project-name> [options]`
Run the autonomous development loop.

```bash
# Without tmux (output directly in terminal)
./ralph/start.sh signals

# With tmux monitoring (recommended)
./ralph/start.sh signals --monitor

# Check status
./ralph/start.sh signals --status

# Reset circuit breaker if stuck
./ralph/start.sh signals --reset
```

Options:
- `-m, --monitor` - Start with tmux session and live monitor
- `-c, --calls NUM` - Max calls per hour (default: 100)
- `-t, --timeout MIN` - Codex timeout in minutes (default: 15)
- `-s, --status` - Show project status and exit
- `-r, --reset` - Reset circuit breaker

### `./ralph/monitor.sh <project-name>`
Live status dashboard (auto-started with `--monitor`).

```bash
./ralph/monitor.sh signals
```

## Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. Write PRD (prd.md)                                          │
│     Human-readable requirements document                        │
│                                                                 │
│  2. Convert to JSON (prd.json)                                  │
│     Codex breaks down PRD into actionable tasks                │
│                                                                 │
│  3. Ralph Loop                                                  │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Pick next story where passes=false                 │     │
│     │  ↓                                                  │     │
│     │  Generate prompt with story + context               │     │
│     │  ↓                                                  │     │
│     │  Run Codex CLI                                     │     │
│     │  ↓                                                  │     │
│     │  Analyze response (check status block)              │     │
│     │  ↓                                                  │     │
│     │  If story complete: mark passes=true, commit        │     │
│     │  ↓                                                  │     │
│     │  If all done: exit                                  │     │
│     │  Else: loop back                                    │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## PRD JSON Format

The `prd.json` file has this structure:

```json
{
  "branchName": "ralph/feature-name",
  "userStories": [
    {
      "id": "1.1",
      "category": "functional",
      "story": "Short description of what to build",
      "steps": [
        "Step 1: What to do",
        "Step 2: Next action",
        "Step 3: How to verify"
      ],
      "acceptance": "Detailed acceptance criteria",
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

### Fields

| Field | Description |
|-------|-------------|
| `branchName` | Git branch Ralph will create/use for this feature |
| `id` | Story identifier (phase.sequence, e.g., "1.1", "2.3") |
| `category` | One of: `technical`, `functional`, `ui` |
| `story` | One-sentence description of what to build |
| `steps` | Actionable steps to complete the story |
| `acceptance` | Definition of "done" |
| `priority` | **Lower = do first** (1-10 MVP, 11-20 Phase 2, 21+ Phase 3) |
| `passes` | Set to `true` when story is complete |
| `notes` | Ralph fills this with learnings during implementation |

## tmux Controls

When running with `--monitor`:

| Keys | Action |
|------|--------|
| `Ctrl+B`, `D` | Detach (keeps running in background) |
| `Ctrl+B`, `←/→` | Switch between panes |
| `Ctrl+B`, `[` | Enter scroll mode (`q` to exit) |
| `tmux ls` | List sessions |
| `tmux attach -t ralph-<project>` | Reattach to session |

## Safety Features

- **Rate Limiting**: Max 100 calls/hour (configurable)
- **Circuit Breaker**: Auto-stops after repeated failures
- **Exit Detection**: Stops when Codex signals completion
- **Branch Isolation**: Each feature runs on its own git branch

## Learnings System

Ralph has a two-tier learning system:

| File | Purpose | Lifetime |
|------|---------|----------|
| `progress.txt` | Session memory for Ralph | Per-project |
| `AGENTS.md` | Permanent docs for humans & future agents | Forever |

### progress.txt Structure

```markdown
## Codebase Patterns
- Migrations: Use IF NOT EXISTS
- Types: Export from actions.ts

## Key Files
- db/schema.ts
- app/auth/actions.ts
---
## 2024-01-15 - Story 1.1
- What was implemented
- **Learnings:** patterns discovered
```

### AGENTS.md Updates

Ralph updates `AGENTS.md` files in directories where it made changes:

✅ **Good additions:**
- "When modifying X, also update Y"
- "This module uses pattern Z"
- "Tests require dev server running"

❌ **Don't add:**
- Story-specific details
- Temporary notes

## Project Structure

```
ralph/
├── new.sh          # Create new project
├── convert.sh      # PRD → JSON converter
├── start.sh        # Main loop
├── monitor.sh      # Status dashboard
├── lib/
│   ├── utils.sh
│   ├── circuit_breaker.sh
│   └── response_analyzer.sh
├── templates/
│   ├── PROMPT.md        # Standard prompt
│   ├── prd-template.md  # PRD template
│   └── prd-schema.json  # JSON example
└── projects/
    └── <your-projects>/
        ├── prd.md         # Your PRD
        ├── prd.json       # Generated tasks
        ├── progress.txt   # Progress log
        ├── status.json    # Current status
        ├── PROMPT.md      # Standard prompt
        └── logs/          # Execution logs
```

## Troubleshooting

### Circuit breaker opened
```bash
./ralph/start.sh <project> --status  # Check what happened
./ralph/start.sh <project> --reset   # Reset and continue
```

### Rate limit hit
Ralph automatically waits for the next hour. You can detach with `Ctrl+B, D` and come back later.

### Codex not responding
Check the logs in `ralph/projects/<project>/logs/` for details.
