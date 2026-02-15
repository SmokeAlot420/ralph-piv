<p align="center">
  <h1 align="center"><b>Ralph PIV</b></h1>
  <p align="center"><i>Plan-Implement-Validate</i></p>
  <p align="center">Solo AI dev workflow for Claude Code. Deep analysis, context-rich plans, independent validation.</p>
</p>

<p align="center">
  <a href="https://github.com/SmokeAlot420/ralph-piv/stargazers"><img src="https://img.shields.io/github/stars/SmokeAlot420/ralph-piv?style=flat-square" alt="GitHub Stars"></a>
  <a href="https://github.com/SmokeAlot420/ralph-piv/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue?style=flat-square" alt="License"></a>
  <img src="https://img.shields.io/badge/Claude_Code-plugin-orange?style=flat-square" alt="Claude Code Plugin">
</p>

<p align="center">
  <code>claude --plugin-dir ./piv</code>
</p>

---

## Why I Built This

I kept running into the same problem: AI agents that half-build things. They'd get 80% there, then miss a requirement, break a test, or just... guess at patterns instead of looking at the codebase.

Ralph PIV fixes that by splitting the work into specialized roles:

- A **researcher** that actually reads your codebase and builds context-rich plans
- An **executor** that implements from those plans
- A **validator** that independently verifies everything (and doesn't trust the executor)
- A **debugger** that fixes what the validator catches

Each agent gets fresh context and specific instructions. No drift, no hallucinations from stale context.

## Who This Is For

- Devs building features who want AI to actually finish the job
- Anyone tired of "it works on my end" from their AI agents
- Vibe coders who want to describe what they want and get working code

## Quick Start

```bash
# Clone
git clone https://github.com/SmokeAlot420/ralph-piv.git

# Use as Claude Code plugin
claude --plugin-dir ./piv

# Build a feature from a PRD
/piv ~/my-project/PRDs/PRD.md 1

# Auto-discover PRD and run all phases
/piv ~/my-project

# Quick single-feature build (no PRD needed)
/mini-piv "add-user-auth"
```

## How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                   RALPH PIV ORCHESTRATOR                       │
├──────────────────────────────────────────────────────────────┤
│  1. Discover / Load PRD                                       │
│  2. Spawn RESEARCHER → codebase analysis + PRP generation     │
│  3. Spawn EXECUTOR → implements PRP requirements              │
│  4. Spawn VALIDATOR → independently verifies everything       │
│  5. If gaps found → DEBUGGER fixes (max 3 cycles)             │
│  6. Commit on PASS, next phase                                │
└──────────────────────────────────────────────────────────────┘
```

1. **Research** — A fresh sub-agent analyzes your codebase, reads docs, and generates a context-rich PRP (Project Requirements Plan)
2. **Execute** — The executor implements from the PRP, following your project's patterns
3. **Validate** — The validator independently checks every requirement against actual code (doesn't trust the executor)
4. **Debug** — If the validator finds gaps, the debugger fixes root causes (max 3 cycles)
5. **Commit** — Clean semantic commit on validation pass

## Why It Works

**Context-rich plans.** The PRP template captures everything: documentation URLs, file patterns, gotchas, validation commands. The executor doesn't guess — it has specific instructions for every task.

**Independent validation.** The validator reads the PRP requirements and checks actual files. If the executor says "I added the function" — the validator greps for it. Trust nothing, verify everything.

**Fresh context per agent.** Each sub-agent spawns with clean context and reads only what it needs. No context drift from long sessions.

## Two Modes

| Mode | When to Use | What It Does |
|------|-------------|--------------|
| `/piv` | Multi-phase projects with a PRD | Iterates through phases, generates PRPs, full pipeline per phase |
| `/mini-piv` | Quick single features, no PRD | Asks 5 discovery questions, generates PRP, builds it |

## Plugin Structure

```
piv/
├── .claude-plugin/plugin.json
├── agents/
│   ├── piv-executor.md
│   ├── piv-validator.md
│   └── piv-debugger.md
└── skills/
    ├── piv/
    │   ├── SKILL.md
    │   ├── references/
    │   │   ├── codebase-analysis.md
    │   │   ├── generate-prp.md
    │   │   ├── execute-prp.md
    │   │   ├── create-prd.md
    │   │   └── piv-discovery.md
    │   └── assets/
    │       ├── prp_base.md
    │       └── workflow-template.md
    └── mini-piv/
        ├── SKILL.md
        ├── references/
        └── assets/
```

## Credits

The **PIV Loop** (Plan-Implement-Validate) was created by [Cole Medin](https://github.com/coleam00) and [Rasmus Widing](https://github.com/rasmuswiding) as part of the [Dynamous Community](https://community.dynamous.ai) [Agentic Coding Course](https://github.com/dynamous-community/agentic-coding-course). Rasmus also created the **PRP (Project Requirements Plan) framework** that Ralph PIV builds on.

Ralph PIV packages their methodology into a Claude Code plugin with automated sub-agent orchestration, independent validation, and targeted debugging loops.

## Related

- **[CLUTCH](https://github.com/SmokeAlot420/clutch)** — Parallel execution with Agent Teams. When your phases have parallelizable work, CLUTCH splits it across 2-4 executor agents working simultaneously.
- **[TeamBox](https://github.com/SmokeAlot420/teambox)** — Real-time dashboard for monitoring your Ralph PIV and CLUTCH agent sessions.
- **[Agentic Coding Course](https://github.com/dynamous-community/agentic-coding-course)** — The course where the PIV Loop and PRP framework originated. By Cole Medin & Rasmus Widing.

## License

MIT
