---
name: mini-piv
description: "Mini PIV — Lightweight single-phase builder. No PRD required, discovery-driven. Quick feature building with the full PIV quality pipeline. Triggers on: mini-piv, quick build, feature, small feature."
disable-model-invocation: true
allowed-tools: Task, TaskCreate, TaskUpdate, TaskList, Read, Write, Bash, Glob, Grep, AskUserQuestion
argument-hint: "[feature-name] [project-path]"
---

# Mini PIV — Lightweight Feature Builder

## Arguments: $ARGUMENTS

Parse arguments:

```
FEATURE_NAME = $ARGUMENTS[0] or null (will ask user during discovery)
PROJECT_PATH = $ARGUMENTS[1] or current working directory
```

**Examples:**
- `/mini-piv` → Asks for feature name during discovery, uses cwd
- `/mini-piv "add-user-auth"` → Feature name provided, uses cwd
- `/mini-piv "token-filters" /home/smoke/projects/myapp` → Both provided

---

## Resolve PIV Plugin Directory

Before proceeding, determine the absolute path to the PIV plugin:
```bash
find /home -maxdepth 4 -name "plugin.json" -path "*/piv/.claude-plugin/*" 2>/dev/null | head -1 | sed 's|/.claude-plugin/plugin.json||'
```
Store the result as `PIV_DIR`. All reference docs and templates are under this directory.

---

## Philosophy: Quick & Quality

> "When you just want to build something without writing a PRD first."

Mini PIV is the lightweight sibling of the full `/piv` orchestrator. Same quality pipeline (Execute → Validate → Debug), but starts from a quick conversation instead of a PRD.

**You are the orchestrator** — stay lean at ~15% context, spawn fresh sub-agents for heavy lifting.

---

## Visual Workflow

```
┌──────────────────────────────────────────────────────────────┐
│              MINI PIV ORCHESTRATOR                             │
├──────────────────────────────────────────────────────────────┤
│ 1. DISCOVERY (Orchestrator)                                   │
│    Ask user 3-5 questions conversationally                    │
│    Extract: scope, touchpoints, deps, success, out-of-scope  │
│                                                               │
│ 2. RESEARCH & PRP GENERATION (Fresh sub-agent)                │
│    ├─ Step 1: Codebase analysis (deep research)               │
│    ├─ Step 2: Generate PRP from discovery answers             │
│    └─ Output: PRPs/mini-{feature-name}.md                     │
│                                                               │
│ 3. EXECUTE (Executor sub-agent)                               │
│    → EXECUTION SUMMARY                                        │
│                                                               │
│ 4. VALIDATE (Validator sub-agent)                             │
│    → PASS / GAPS_FOUND / HUMAN_NEEDED                         │
│                                                               │
│ 5. DEBUG LOOP (if needed, max 3 iterations)                   │
│    Spawn DEBUGGER → Re-validate → repeat or escalate          │
│                                                               │
│ 6. COMMIT (Orchestrator)                                      │
│    feat(mini): {description}                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1: Discovery Phase

### 1a. Determine Feature Name

If `FEATURE_NAME` is not provided in arguments:
1. Check recent conversation for feature context
2. If clear from context, propose a name and confirm
3. If unclear, ask the user directly

Normalize the name to kebab-case (e.g., "User Authentication" → "user-authentication").

### 1b. Check for Existing PRP

```bash
ls -la PROJECT_PATH/PRPs/ 2>/dev/null | grep -i "mini-{FEATURE_NAME}"
```

If `PRPs/mini-{FEATURE_NAME}.md` already exists:
- Tell the user and ask: "A PRP for '{FEATURE_NAME}' already exists. Would you like to overwrite it, use a different name, or skip straight to execution?"

### 1c. Ask Discovery Questions (Conversational)

Present all questions in a single conversational message. Adapt the questions based on what you already know from the feature name and context.

**Default question set:**

```
I've got a few quick questions so I can build this right:

1. **What does this feature do?** Give me the quick rundown — what's it supposed to accomplish?

2. **Where in the codebase does it live?** Any specific files, folders, or components it touches? (Say "not sure" if you want me to figure it out)

3. **Any specific libraries, patterns, or existing code to follow?** (e.g., "use the existing useAuth hook" or "follow the pattern in src/api/users.ts")

4. **What does "done" look like?** How will we know it's working? Give me 1-3 concrete success criteria.

5. **Anything explicitly OUT of scope?** What should I definitely NOT build right now?
```

**Adapt questions by feature type:**

For **UI Components** — also ask about styling, props, responsive behavior
For **API Endpoints** — also ask about request/response format, auth requirements
For **Smart Contracts** — also ask about functions, events, security requirements
For **Integrations** — also ask about external service details, error handling

**Wait for user response**, then extract structured answers.

### 1d. Structure Discovery Answers

After the user responds, extract their answers into this format for the research agent:

```yaml
feature:
  name: {FEATURE_NAME}
  scope: {Answer to question 1 - what it does}
  touchpoints: {Answer to question 2 - where it lives}
  dependencies: {Answer to question 3 - libraries/patterns}
  success_criteria: {Answer to question 4 - what done looks like}
  out_of_scope: {Answer to question 5 - what to skip}
```

---

## Step 2: Research & PRP Generation

**CRITICAL: Do NOT generate the PRP yourself as the orchestrator. Spawn a FRESH sub-agent.**

Spawn a `general-purpose` sub-agent with this prompt:

```
MINI PIV: RESEARCH & PRP GENERATION
==========================================

You are generating a PRP for a lightweight, single-phase feature. You have fresh context — use it wisely.

Project root: {PROJECT_PATH}
Feature name: {FEATURE_NAME}

## Discovery Input (from user)
{paste structured YAML from Step 1d}

## Step 1: Codebase Analysis
Read and follow {PIV_DIR}/skills/mini-piv/references/codebase-analysis.md

Run deep codebase analysis for this feature:
- Focus area: {feature scope from discovery}
- Touch points: {touchpoints from discovery}
- Dependencies/patterns: {dependencies from discovery}

Save analysis to: {PROJECT_PATH}/PRPs/planning/mini-{FEATURE_NAME}-analysis.md

## Step 2: Generate PRP (with analysis context still loaded)
Read and follow {PIV_DIR}/skills/mini-piv/references/generate-prp.md
Read the PRP template at: {PIV_DIR}/skills/mini-piv/assets/prp_base.md

You already have the codebase analysis in your context from Step 1 — use it directly.
DO NOT spawn a sub-agent for PRP generation. You do it yourself.

### Discovery → PRP Translation Guide

Map the user's discovery answers to PRP template sections:

| Discovery Answer | PRP Section |
|-----------------|-------------|
| Feature Scope (Q1) | Goal + What |
| Touch Points (Q2) | Implementation Blueprint task locations |
| Dependencies (Q3) | Context YAML → patterns, Known Gotchas |
| Success Criteria (Q4) | Success Criteria checklist + Final Validation |
| Out of Scope (Q5) | Explicit exclusions in What section |

Output PRP to: {PROJECT_PATH}/PRPs/mini-{FEATURE_NAME}.md

## Critical Rules
- Do BOTH steps yourself in sequence
- Follow the full generate-prp process (template, quality gates, info density)
- DO NOT spawn sub-agents for either step
```

**Wait for the research agent to complete** before proceeding.

---

## Step 3: Spawn Executor Sub-Agent

Use the Task tool with `subagent_type: "piv-executor"`:

```
EXECUTOR MISSION - Mini PIV
============================

Read the PRP execution process doc at: {PIV_DIR}/skills/mini-piv/references/execute-prp.md

PRP Path: {PROJECT_PATH}/PRPs/mini-{FEATURE_NAME}.md
Project: {PROJECT_PATH}

Follow: Load PRP → Think Deeply & Plan → Execute → Validate → Verify
Output EXECUTION SUMMARY with Status, Files, Tests, Issues.
```

**Wait for executor to complete.**

---

## Step 4: Spawn Validator Sub-Agent

Use the Task tool with `subagent_type: "piv-validator"`:

```
VALIDATOR MISSION - Mini PIV
=============================

PRP Path: {PROJECT_PATH}/PRPs/mini-{FEATURE_NAME}.md
Project: {PROJECT_PATH}
Executor Summary:
{paste EXECUTION SUMMARY from Step 3}

Verify ALL requirements independently. Don't trust executor claims.
Output VERIFICATION REPORT with Grade (PASS/GAPS_FOUND/HUMAN_NEEDED), Checks, Gaps.
```

**Process result:**
- `PASS` → Proceed to Step 6 (commit)
- `GAPS_FOUND` → Proceed to Step 5 (debug)
- `HUMAN_NEEDED` → Ask user for guidance

---

## Step 5: Debug Loop (Max 3 iterations)

Spawn debugger using `subagent_type: "piv-debugger"`:

```
DEBUGGER MISSION - Mini PIV - Iteration {I}
============================================

Project: {PROJECT_PATH}
PRP Path: {PROJECT_PATH}/PRPs/mini-{FEATURE_NAME}.md
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

---

## Step 6: Smart Commit (Orchestrator does this)

After validation passes:

```bash
cd PROJECT_PATH
git status
git diff --stat
```

Create semantic commit:

```bash
git add -A
git commit -m "$(cat <<'EOF'
feat(mini): implement {FEATURE_NAME}

- {bullet point 1 - key thing built}
- {bullet point 2 - key thing built}
- {bullet point 3 - key thing built}

Built with PIV - https://github.com/SmokeAlot420/piv
EOF
)"
```

---

## Completion Summary

```
## MINI PIV COMPLETE

Feature: {FEATURE_NAME}
Project: {PROJECT_PATH}

### Artifacts
- PRP: PRPs/mini-{FEATURE_NAME}.md
- Analysis: PRPs/planning/mini-{FEATURE_NAME}-analysis.md

### Implementation
- Validation cycles: {N}
- Debug iterations: {M}

### Files Changed
{list of created/modified files}

All requirements verified and passing.
```

---

## Error Handling

### Executor Returns BLOCKED
Ask user: "Executor blocked on '{FEATURE_NAME}'. Issue: {description}. How should we proceed?"

### Validator Returns HUMAN_NEEDED
Ask user: "Validator needs guidance on '{FEATURE_NAME}'. Question: {details}. Please advise."

### 3 Debug Cycles Exhausted
Ask user: "'{FEATURE_NAME}' failed validation after 3 fix attempts. Persistent issues: {list}. Need your guidance."
