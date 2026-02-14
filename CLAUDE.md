# PIV — Plan-Implement-Validate

Solo AI dev workflow plugin for Claude Code. All-markdown — skill definitions, agent defs, process docs, and templates. No build system, no tests, no application code.

## Architecture

```
piv/
├── .claude-plugin/plugin.json       <- Plugin metadata
├── agents/                          <- Agent definitions
│   ├── piv-executor.md              <- Implements PRP requirements
│   ├── piv-validator.md             <- Verifies all PRP requirements met
│   └── piv-debugger.md              <- Fixes validator-identified gaps
├── skills/
│   ├── piv/                         <- The /piv skill
│   │   ├── SKILL.md                 <- Main orchestrator
│   │   ├── references/              <- Process docs
│   │   │   ├── piv-discovery.md     <- Discovery questions for new projects
│   │   │   ├── create-prd.md        <- PRD generation process
│   │   │   ├── codebase-analysis.md <- Deep codebase analysis process
│   │   │   ├── generate-prp.md      <- PRP generation process
│   │   │   └── execute-prp.md       <- PRP execution process
│   │   └── assets/                  <- Templates
│   │       ├── prp_base.md          <- PRP template (copied to user project)
│   │       └── workflow-template.md <- WORKFLOW.md tracker template
│   └── mini-piv/                    <- The /mini-piv skill
│       ├── SKILL.md                 <- Lightweight single-feature orchestrator
│       ├── references/              <- Process docs (subset)
│       └── assets/                  <- Templates (subset)
├── CLAUDE.md                        <- This file
├── README.md                        <- Public docs
└── LICENSE                          <- MIT
```

## How It Works

1. **Discover** — If no PRD exists, ask discovery questions and generate one
2. **Analyze** — Fresh sub-agent does deep codebase analysis + PRP generation
3. **Execute** — Executor sub-agent implements PRP requirements
4. **Validate** — Independent validator checks all work
5. **Debug** — Debugger fixes gaps (max 3 cycles)
6. **Commit** — Orchestrator commits on PASS

Two modes:
- `/piv` — Multi-phase, PRD-driven workflow
- `/mini-piv` — Single-phase, discovery-driven, no PRD needed

## Key Mechanism: PIV_DIR Resolution

SKILL.md resolves the plugin's install path at runtime via `find`, then passes absolute paths to all sub-agents:
- `{PIV_DIR}/skills/piv/references/codebase-analysis.md`
- `{PIV_DIR}/skills/piv/references/generate-prp.md`
- `{PIV_DIR}/skills/piv/assets/prp_base.md`

This ensures fresh sub-agents (which have no inherited context) can locate process docs and templates.

## Claude Code Conventions

### Skill Frontmatter

```yaml
---
name: piv
description: "PIV — Plan-Implement-Validate..."
disable-model-invocation: true
allowed-tools: Task, TaskCreate, TaskUpdate, TaskList, Read, Write, Bash, Glob, Grep
argument-hint: "[PRD_PATH|PROJECT_PATH] [START_PHASE] [END_PHASE]"
---
```

### Agent Frontmatter

```yaml
---
name: piv-executor
description: Executor - implements PRP requirements with fresh context.
tools: Read, Write, Edit, Bash, Glob, Grep
model: inherit
---
```

### Plugin Structure

- Only `plugin.json` goes in `.claude-plugin/`
- Agents, skills, references, and assets go at plugin root

## Rules

- **SKILL.md ~500 lines** — heavy content goes in `references/`
- **Templates in `assets/`** — PRP template, workflow template
- **Sub-agents get fresh context** — orchestrator stays lean (~15% context)
- **Sub-agents get absolute paths** — all reference/template paths use `{PIV_DIR}` prefix
- **Flat agent hierarchy** — sub-agents cannot spawn sub-agents
- **PIV branding in commits** — `Built with PIV - https://github.com/SmokeAlot420/piv`

## Usage

```bash
claude --plugin-dir ~/piv
```

Then:
- `/piv [PRD_PATH|PROJECT_PATH] [START_PHASE] [END_PHASE]`
- `/mini-piv [feature-name] [project-path]`
