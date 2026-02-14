---
name: piv
description: "PIV — Plan-Implement-Validate. Solo AI dev workflow with deep analysis, context-rich plans, and independent validation. Spawns specialized sub-agents for execution, validation, and debugging. Triggers on: piv, build, implement, execute."
disable-model-invocation: true
allowed-tools: Task, TaskCreate, TaskUpdate, TaskList, Read, Write, Bash, Glob, Grep
argument-hint: "[PRD_PATH|PROJECT_PATH] [START_PHASE] [END_PHASE]"
---

# PIV Orchestrator

> Plan-Implement-Validate

## Arguments: $ARGUMENTS

Parse arguments using this logic:

### PRD Path Mode (first argument ends with `.md`)

If the first argument ends with `.md`, it's a direct path to a PRD file:
- `PRD_PATH` - Direct path to the PRD file
- `PROJECT_PATH` - Derived by going up from PRDs/ folder
- `START_PHASE` - Second argument (default: 1)
- `END_PHASE` - Third argument (default: auto-detect from PRD)

### Project Path Mode

If the first argument does NOT end with `.md`:
- `PROJECT_PATH` - Absolute path to project (default: current working directory)
- `START_PHASE` - Second argument (default: 1)
- `END_PHASE` - Third argument (default: 4)
- `PRD_PATH` - Auto-discover from `PROJECT_PATH/PRDs/` folder

### Detection Logic

```
If $ARGUMENTS[0] ends with ".md":
  PRD_PATH = $ARGUMENTS[0]
  PROJECT_PATH = dirname(dirname(PRD_PATH))
  START_PHASE = $ARGUMENTS[1] or 1
  END_PHASE = $ARGUMENTS[2] or auto-detect from PRD
  PRD_NAME = basename without extension
Else:
  PROJECT_PATH = $ARGUMENTS[0] or current working directory
  START_PHASE = $ARGUMENTS[1] or 1
  END_PHASE = $ARGUMENTS[2] or 4
  PRD_PATH = auto-discover from PROJECT_PATH/PRDs/
  PRD_NAME = discovered PRD basename
```

### Mode Detection

After parsing arguments:
- If PRD_PATH was provided or auto-discovered → **MODE = "execute"**
- If no PRD found → **MODE = "discovery"**

### Auto-Detect Phases from PRD

When PRD_PATH is specified, scan the PRD for phase sections:
1. Look for: `## Phase N:`, `### Phase N:`, `**Phase N:**`, `Phase N:`
2. Set END_PHASE to highest phase found (if not specified)

---

## Resolve PIV Plugin Directory

Before proceeding, determine the absolute path to the PIV plugin:
```bash
# Find the PIV plugin root (parent of .claude-plugin/)
find /home -maxdepth 4 -name "plugin.json" -path "*/piv/.claude-plugin/*" 2>/dev/null | head -1 | sed 's|/.claude-plugin/plugin.json||'
```
Store the result as `PIV_DIR`. All reference docs and templates are under this directory.

Verify: `PIV_DIR/skills/piv/assets/prp_base.md` should exist.

---

## Required Reading by Role

**CRITICAL: Each role MUST read their instruction files before acting.**

| Role | Instructions |
|------|-------------|
| Discovery (no PRD) | Read {PIV_DIR}/skills/piv/references/piv-discovery.md |
| PRD Creation | Read {PIV_DIR}/skills/piv/references/create-prd.md |
| PRP Generation | Read {PIV_DIR}/skills/piv/references/generate-prp.md |
| Executor | Read {PIV_DIR}/skills/piv/references/execute-prp.md |

**DO NOT wing it. Follow the established processes.**

**Prerequisite:** A PRD must exist before entering the Phase Workflow. If no PRD exists, enter Discovery Mode.

---

## Discovery Mode (No PRD Found)

When MODE = "discovery":

1. Read {PIV_DIR}/skills/piv/references/piv-discovery.md for the discovery process
2. Present discovery questions to the user in a friendly, conversational tone
   - Target audience is vibe coders — keep it approachable
   - Skip questions the user already answered
3. Wait for user answers
4. Fill gaps with your expertise:
   - If user doesn't know tech stack → research and PROPOSE one
   - If user can't define phases → propose 3-4 phases based on scope
   - Always propose-and-confirm: "Here's what I'd suggest — does this sound right?"
5. Run project setup:
   - Create directories: PRDs/, PRPs/templates/, PRPs/planning/
   - Copy PRP template: `cp {PIV_DIR}/skills/piv/assets/prp_base.md {PROJECT_PATH}/PRPs/templates/prp_base.md`
6. Generate PRD: Read {PIV_DIR}/skills/piv/references/create-prd.md, write to PROJECT_PATH/PRDs/PRD-{project-name}.md
7. Set PRD_PATH to the generated PRD, auto-detect phases → continue to Phase Workflow

The orchestrator handles discovery and PRD generation directly (no sub-agent needed).

---

## Orchestrator Philosophy

> "Context budget: ~15% orchestrator, 100% fresh per sub-agent"

You are the **orchestrator**. You stay lean and manage workflow. You DO NOT execute PRPs yourself — you spawn specialized sub-agents with fresh context for each task.

---

## Phase Workflow

For each phase from START_PHASE to END_PHASE:

### Step 1: Check/Generate PRP

#### Step 1a: Check for existing PRP
```bash
ls -la PROJECT_PATH/PRPs/ 2>/dev/null | grep -i "phase.*N\|pN\|p-N"
```
If a PRP already exists for this phase, skip to Step 2.

#### Step 1b: Spawn Fresh Research Agent for PRP Generation

**CRITICAL: Do NOT generate the PRP yourself. Spawn a FRESH sub-agent.**

Before spawning, the orchestrator must:
1. Read the PRD at PRD_PATH
2. Find the Phase N section
3. Extract the phase scope (title, deliverables, validation criteria)
4. Pass this extracted scope to the fresh agent

Spawn a `general-purpose` sub-agent with this prompt:

```
RESEARCH & PRP GENERATION MISSION - Phase {N}
==============================================

You are generating a PRP for Phase {N}. You have fresh context — use it wisely.

Project root: {PROJECT_PATH}
PRD Path: {PRD_PATH}

## Phase {N} Scope (from PRD)
{paste phase title, deliverables, and validation criteria}

## Step 1: Codebase Analysis
Read the codebase analysis process doc at: {PIV_DIR}/skills/piv/references/codebase-analysis.md
Follow it fully. Run deep codebase analysis for: {phase feature description}
Save analysis to: {PROJECT_PATH}/PRPs/planning/{PRD_NAME}-phase-{N}-analysis.md

## Step 2: Generate PRP (analysis context still loaded)
Read the PRP generation process doc at: {PIV_DIR}/skills/piv/references/generate-prp.md
Read the PRP template at: {PIV_DIR}/skills/piv/assets/prp_base.md
Follow the process doc fully. Use the template structure.
You already have the codebase analysis in your context — use it directly.
DO NOT spawn a sub-agent for this. You do it yourself.
Output PRP to: {PROJECT_PATH}/PRPs/PRP-{PRD_NAME}-phase-{N}.md

## Critical Rules
- Do BOTH steps yourself in sequence
- Your analysis context feeds directly into PRP quality
- Follow the full generate-prp process (template, quality gates, info density)
- The PRP template is at: {PIV_DIR}/skills/piv/assets/prp_base.md
- DO NOT spawn sub-agents for either step
```

**Wait for the research agent to complete** before proceeding.

### Step 2: Spawn Executor Sub-Agent

Use the Task tool with `subagent_type: "piv-executor"`:

```
EXECUTOR MISSION - Phase {N}
============================

Read the PRP execution process doc at: {PIV_DIR}/skills/piv/references/execute-prp.md

PRP Path: {PRP_PATH}
Project: {PROJECT_PATH}

Follow: Load PRP → Think Deeply & Plan → Execute → Validate → Verify
Output EXECUTION SUMMARY with Status, Files, Tests, Issues.
```

**Wait for executor to complete** before proceeding.

### Step 3: Spawn Validator Sub-Agent

Use the Task tool with `subagent_type: "piv-validator"`:

```
VALIDATOR MISSION - Phase {N}
=============================

PRP Path: {PRP_PATH}
Project: {PROJECT_PATH}
Executor Summary:
{paste EXECUTION SUMMARY here}

Verify ALL requirements independently. Don't trust executor claims.
Check every file, every test, every requirement in the PRP.
Output VERIFICATION REPORT with Grade (PASS/GAPS_FOUND/HUMAN_NEEDED), Checks, Gaps.
```

**Process validator result:**
- `PASS` → Proceed to Step 5 (commit)
- `GAPS_FOUND` → Proceed to Step 4 (debug)
- `HUMAN_NEEDED` → Ask user for guidance

### Step 4: Debug Loop (Max 3 iterations)

Spawn debugger sub-agent using `subagent_type: "piv-debugger"`:

```
DEBUGGER MISSION - Phase {N} - Iteration {I}
============================================

Project: {PROJECT_PATH}
PRP Path: {PRP_PATH}
Gaps: {GAPS}
Errors: {ERRORS}

Fix root causes, not symptoms. Run tests after each fix.
Output FIX REPORT with Status, Fixes Applied, Test Results.
```

After fixes:
- Re-run validator
- If PASS → proceed to commit
- If GAPS_FOUND again → debug again (up to 3 total)
- After 3 iterations → escalate to user

### Step 5: Smart Commit (Orchestrator does this)

After validation passes:
```bash
cd PROJECT_PATH
git status
git diff --stat
```

Create semantic commit:
- Format: `feat/fix/refactor(scope): description`
- Add: `Built with PIV - https://github.com/SmokeAlot420/piv`

### Step 6: Update Progress

Update `PROJECT_PATH/WORKFLOW.md`:
- Mark phase N as complete
- Note validation results
- Record any issues or observations

### Step 7: Next Phase

Increment phase counter. If more phases remain, loop back to Step 1.

---

## Error Handling

### No PRD Found
Enter Discovery Mode (see above).

### Executor Returns BLOCKED
Ask user: "Executor blocked on phase N. Issue: [description]. How should we proceed?"

### Validator Returns HUMAN_NEEDED
Ask user: "Validator needs guidance on phase N. Question: [details]. Please advise."

### 3 Debug Cycles Exhausted
Ask user: "Phase N failed validation after 3 fix attempts. Persistent issues: [list]. Need your guidance."

---

## Completion

When all phases are complete, output:
```
## PIV COMPLETE

Phases Completed: START to END
Total Commits: N
Validation Cycles: M

### Phase Summary:
- Phase 1: [feature] - validated in N cycles
- Phase 2: [feature] - validated in N cycles
...

All phases successfully implemented and validated.
```

---

## Visual Workflow

```
┌──────────────────────────────────────────────────────────────┐
│                    PIV ORCHESTRATOR                            │
│              Plan-Implement-Validate                          │
├──────────────────────────────────────────────────────────────┤
│ IF NO PRD FOUND:                                              │
│   a. Ask discovery questions (piv-discovery.md)               │
│   b. Generate PRD from answers (create-prd.md)                │
│   c. Set PRD_PATH, auto-detect phases                         │
│                                                               │
│ FOR EACH PHASE (START_PHASE to END_PHASE):                    │
│   a. Check if PRP exists                                      │
│   b. If not → spawn RESEARCH AGENT (analysis + PRP gen)       │
│   c. Spawn EXECUTOR → returns EXECUTION SUMMARY               │
│   d. Spawn VALIDATOR → returns PASS/GAPS_FOUND/HUMAN_NEEDED   │
│   e. If GAPS_FOUND → Spawn DEBUGGER (max 3x)                  │
│   f. Commit on PASS                                           │
│   g. Update WORKFLOW.md                                       │
│   h. Next phase                                               │
└──────────────────────────────────────────────────────────────┘
```
