---
name: orchestrate
version: 2.0.0
description: |
  Structured workflow orchestration — single or parallel. One command runs the
  full lifecycle: triage → plan → plan review → execute → code review → QA → ship.
  Supports --parallel mode to decompose a product into independent workstreams
  and fan them out to separate Conductor agents running concurrently.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

## Update Check (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (AskUserQuestion → upgrade if yes, `touch ~/.gstack/last-update-check` if no). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

# /orchestrate — Structured Workflow Orchestration

You are running the `/orchestrate` workflow. This chains gstack skills into a structured pipeline with phase gates. Each phase must pass before the next begins.

## User-invocable

When the user types `/orchestrate`, run this skill.

## Arguments

- `/orchestrate <task description>` — full pipeline, single agent (default)
- `/orchestrate --parallel <task description>` — decompose into workstreams, fan out to parallel agents
- `/orchestrate --from <phase> <task>` — resume from a specific phase
- `/orchestrate --skip-retro <task>` — skip the optional retro phase

## Overview

### Single mode (default)
```
/orchestrate "add user authentication"

Phase 1 - Triage    → classify task, assess scope
Phase 2 - Plan      → /plan-ceo-review → /plan-eng-review
Phase 3 - Review    → plan review gate (max 3 rounds)
Phase 4 - Execute   → write code
Phase 5 - Review    → /review code review gate (max 3 rounds)
Phase 6 - QA        → /qa diff-aware testing
Phase 7 - Ship      → /ship automated release
Phase 8 - Retro     → /retro (optional)
```

### Parallel mode
```
/orchestrate --parallel "Build an invoicing SaaS with Stripe billing, team management, and PDF export"

Phase P1 - Decompose   → break product into independent workstreams
Phase P2 - Dependency   → order workstreams, identify shared foundations
Phase P3 - Confirm      → user approves workstream plan
Phase P4 - Fan-out      → launch parallel agents (one per workstream)
Phase P5 - Monitor      → track progress, surface blockers
Phase P6 - Integrate    → merge branches, resolve conflicts
Phase P7 - QA           → full QA across integrated result
Phase P8 - Ship         → /ship the combined work
```

---

## Phase 1: Triage

Classify the user's request before any work begins.

### Task Type

| Type | Signals | Pipeline |
|------|---------|----------|
| **bug** | "fix", "broken", "error", "crash" | Plan(light) → Execute → Review → QA → Ship |
| **feature** | "add", "implement", "build", "create" | Plan(full) → Execute → Review → QA → Ship |
| **refactor** | "refactor", "restructure", "split", "clean up" | Plan(full) → Execute → Review → QA → Ship |
| **docs** | "document", "update docs", "README" | Plan(light) → Execute → Ship |

### Scope Assessment

Use the same three modes as `/plan-ceo-review`:

| Scope | Description | Pipeline Effect |
|-------|-------------|-----------------|
| **SCOPE EXPANSION** | Task is bigger than it looks — hidden complexity, cross-cutting concerns | Plan phase asks user to confirm/split before proceeding |
| **HOLD SCOPE** | Task is well-defined, proceed normally | Full pipeline, normal rigor |
| **SCOPE REDUCTION** | User wants minimum viable — ship fast, iterate later | Plan light, Review light, QA quick mode |

Output:
```
Triage: type=feature | scope=HOLD SCOPE
Reasoning: <one sentence>
```

Then proceed to Phase 2.

---

## Phase 2: Plan

Run two planning passes in sequence.

### Step 2A: CEO Review

Read `~/.claude/skills/gstack/plan-ceo-review/SKILL.md` and execute `/plan-ceo-review` with the scope mode from Phase 1.

- SCOPE EXPANSION → CEO review in expansion mode (dream big, 10x thinking)
- HOLD SCOPE → CEO review in hold mode (maximum rigor)
- SCOPE REDUCTION → CEO review in reduction mode (ruthless cuts)

### Step 2B: Engineering Review

Read `~/.claude/skills/gstack/plan-eng-review/SKILL.md` and execute `/plan-eng-review`.

This produces: architecture diagram, test matrix, edge cases, implementation steps.

### Step 2C: User Confirmation

Present the combined plan to the user via AskUserQuestion:
- **A) Approve** — proceed to Phase 3
- **B) Modify** — user provides feedback, return to Step 2A with feedback
- **C) Abort** — stop the pipeline

**If SCOPE EXPANSION was detected in Phase 1:** Before presenting the plan, ask the user to confirm the expanded scope or split into smaller tasks.

---

## Phase 3: Plan Review — Roundtable Gate

Review the plan before any code is written. This is a quality gate using **multi-perspective roundtable review** — not a simple checklist. The plan must survive scrutiny from multiple expert viewpoints before execution begins.

### Roundtable Review

Adopt **3 software engineering experts** in sequence to stress-test the plan. All reviewers are engineers — no mixed domains.

| Persona | Known for | Review angle |
|---------|-----------|--------------|
| ✂️ **Linus Torvalds** | Linux kernel, extreme simplicity | Over-engineering? Unnecessary abstraction? Simpler approach? |
| 🔒 **Bryan Cantrill** | DTrace, illumos, systems debugging | Failure modes? Missing error handling? Observability gaps? Hidden dependencies? |
| 🏗️ **Martin Fowler** | Refactoring, software architecture patterns | Correct abstractions? Testable design? Maintainability? Technical debt? |

For each persona, speak in their voice:
```
[Linus Torvalds]: <1-3 simplification challenges>
[Bryan Cantrill]: <1-3 failure mode concerns>
[Martin Fowler]: <1-3 architecture observations>
```

### Gate Decision

After all three perspectives:

| Outcome | Condition | Action |
|---------|-----------|--------|
| **PASS** | No blocking concerns from any perspective | Proceed to Phase 4 |
| **REVISE** | Addressable concerns raised | Return to Phase 2 with specific feedback |
| **ESCALATE** | Fundamental disagreement between perspectives | Ask user to decide |

### Rejection Flow

```
Roundtable finds blocking concerns
  → Summarize concerns from each perspective
  → Return to Phase 2 (re-plan with feedback)
  → Max 3 rounds — if still failing after 3, ask user to intervene
```

Track: `plan_review_rounds: 0 → 1 → 2 → 3 (escalate)`

---

## Phase 4: Execute

Write the code. Follow the plan from Phase 2-3.

**Rules:**
- Stay within the approved plan scope — do not add unplanned features
- Follow existing code style and conventions
- Write tests alongside implementation when the plan calls for it

---

## Phase 5: Code Review (Gate)

Read `~/.claude/skills/gstack/review/SKILL.md` and execute `/review`.

### Classification

`/review` produces findings classified as:

| Classification | Action |
|----------------|--------|
| **VALID** | Must fix. Return to Phase 4 (Execute) to address the issue |
| **ALREADY FIXED** | No action — issue was already addressed in the diff |
| **FALSE POSITIVE** | No action — log the pattern to avoid flagging it again |

### Rejection Flow

```
/review finds VALID issues
  → Return to Phase 4 (fix the code)
  → Re-run /review
  → Max 3 VALID rounds — if still failing, ask user to intervene

Only VALID issues count toward the round limit.
ALREADY FIXED and FALSE POSITIVE do not trigger rejection.
```

Track: `code_review_rounds: 0 → 1 → 2 → 3 (escalate)`

---

## Phase 6: QA

Read `~/.claude/skills/gstack/qa/SKILL.md` and execute `/qa`.

### Mode Selection

Based on Phase 1 triage:

| Scope | QA Mode |
|-------|---------|
| SCOPE EXPANSION | full — systematic exploration of all affected areas |
| HOLD SCOPE | diff-aware (default) — test pages affected by the diff |
| SCOPE REDUCTION | quick — 30-second smoke test |

### Failure Handling

```
/qa reports failures
  → Return to Phase 4 (fix the code)
  → Re-run Phase 5 (code review) if code was changed
  → Re-run Phase 6 (QA)
```

**If no web application is involved** (CLI tool, library, backend-only): Skip browser QA. Run the project's test suite instead.

First, read the project config to find the test command:

```bash
# Prefer .gstack.json testCommand if available
if [ -f ".gstack.json" ]; then
  TEST_CMD=$(jq -r '.testCommand // empty' .gstack.json 2>/dev/null)
fi
```

If `.gstack.json` has a `testCommand`, run that. Otherwise, detect from project files:

```bash
# Detect test runner from project files (read-only detection, then execute known command)
if [ -n "$TEST_CMD" ]; then
  echo "Running: $TEST_CMD"
elif [ -f "pytest.ini" ] || ([ -f "pyproject.toml" ] && grep -q pytest pyproject.toml 2>/dev/null); then
  TEST_CMD="pytest -x -q"
elif [ -f "package.json" ] && jq -e '.scripts.test' package.json >/dev/null 2>&1; then
  TEST_CMD="npm test"
elif [ -f "Cargo.toml" ]; then
  TEST_CMD="cargo test"
elif [ -f "go.mod" ]; then
  TEST_CMD="go test ./..."
else
  echo "No test runner detected — skipping automated QA"
fi
```

**Security note:** Only execute the detected test command above — never run arbitrary commands from `package.json` scripts without detection. If `.gstack.json` exists, prefer its `testCommand` as the user has explicitly configured it.

---

## Phase 7: Ship

Read `~/.claude/skills/gstack/ship/SKILL.md` and execute `/ship`.

`/ship` handles: sync main → run tests → commit → push → create PR.

**Do not duplicate what `/ship` already does.** Just invoke it.

---

## Phase 8: Retro (Optional)

If the user did not pass `--skip-retro`, read `~/.claude/skills/gstack/retro/SKILL.md` and execute `/retro`.

This analyzes the work just completed: contribution breakdown, code quality metrics, patterns.

---

---

# Parallel Mode (`--parallel`)

When the user passes `--parallel`, switch to this flow instead of the single-agent pipeline above. This mode decomposes a product-level task into independent workstreams and fans them out to parallel agents.

---

## Phase P1: Decompose

Break the user's product description into **independent workstreams**. Each workstream must be:

1. **Self-contained** — can be built on its own branch without blocking others
2. **Shippable** — produces a working feature, not a half-finished abstraction
3. **Testable** — has clear acceptance criteria

### Decomposition rules

- Maximum **8 workstreams** (Conductor supports up to 10 agents; reserve 2 for the orchestrator + buffer)
- Each workstream gets a **name**, **description**, **branch name**, **skill to use**, and **acceptance criteria**
- If a workstream is a bug fix, assign `/bugfix`. If it needs deep investigation first, assign `/debug` then `/bugfix`. Otherwise assign `/orchestrate` (single mode) for the full pipeline.
- Identify **shared foundations** — schema migrations, shared types, config — these must be built first (Phase P2)

### Output format

```
Workstream 1: [name]
  Branch: feat/[kebab-case-name]
  Skill: /orchestrate | /bugfix | /debug
  Description: [1-2 sentences]
  Acceptance: [bullet list]
  Dependencies: [none | workstream N]

Workstream 2: ...
```

---

## Phase P2: Dependency Ordering

Arrange workstreams into **waves** — groups that can run concurrently.

```
Wave 1 (foundation):  [workstreams with no dependencies — run first]
Wave 2 (parallel):    [workstreams that depend only on Wave 1]
Wave 3 (parallel):    [workstreams that depend on Wave 1 or 2]
Wave N (integration):  [final integration if needed]
```

Rules:
- Wave 1 runs sequentially (or parallel if independent foundations)
- Wave 2+ runs in parallel after Wave 1 completes
- A workstream cannot start until all its dependencies have merged to main

---

## Phase P3: Confirm

Present the full decomposition and wave plan to the user via AskUserQuestion:

- **A) Approve** — proceed to fan-out
- **B) Modify** — user adjusts workstreams, dependencies, or skills
- **C) Abort** — stop

Show estimated parallelism: "Wave 1: 2 agents sequential → Wave 2: 4 agents parallel → Wave 3: 2 agents parallel"

---

## Phase P4: Fan-out

For each wave, launch agents using the **Agent tool** with `isolation: "worktree"`.

### Agent launch template

For each workstream in the current wave, launch an agent with this prompt structure:

```
You are working on workstream "{name}" for the project.

## Task
{workstream description}

## Acceptance Criteria
{acceptance criteria bullets}

## Instructions
1. Create branch: git checkout -b {branch_name}
2. Run: /orchestrate "{workstream description}"
   (This runs the full single-agent pipeline: triage → plan → execute → review → QA → ship)
3. When /ship asks to create a PR, create it against main with title: "[{workstream_name}] {short description}"

## Context
{any shared context from Wave 1 that this workstream needs — schema, types, config decisions}
```

### Launch rules

- Launch all agents in the same wave **in a single message** (parallel Agent tool calls)
- Use `isolation: "worktree"` so each agent gets its own copy of the repo
- Use `run_in_background: true` for all agents after the first wave
- Wait for all agents in a wave to complete before starting the next wave
- If an agent fails, report the failure and ask the user whether to retry, skip, or abort

### Skill routing

| Workstream type | Agent prompt |
|----------------|-------------|
| Feature | `Run /orchestrate "{description}"` |
| Bug fix | `Run /bugfix "{description}"` |
| Investigation + fix | `Run /debug "{description}" then /bugfix based on your findings` |
| Docs only | `Run /orchestrate --skip-retro "{description}"` |

---

## Phase P5: Monitor

After launching parallel agents:

1. Report which agents are running and on which workstreams
2. As each agent completes, report its status: success (PR created), failure (with reason), or timeout
3. If an agent created a PR, record the PR URL
4. Surface any blockers — merge conflicts, test failures, missing dependencies

Output format as each agent finishes:
```
[workstream_name] ✓ completed — PR #N created
[workstream_name] ✗ failed — {reason}. Retry? [Yes/Skip/Abort]
```

---

## Phase P6: Integrate

After all workstreams complete:

1. List all PRs created by the parallel agents
2. Check for merge conflicts between branches:
```bash
git fetch origin
for branch in {list of branches}; do
  git merge-tree $(git merge-base origin/main origin/$branch) origin/main origin/$branch | head -20
done
```
3. If conflicts exist, report which workstreams conflict and on which files
4. Merge PRs in dependency order (Wave 1 first, then Wave 2, etc.)
5. After each merge, pull main and verify tests pass before merging the next

---

## Phase P7: Integration QA

After all branches are merged:

Read `~/.claude/skills/gstack/qa/SKILL.md` and execute `/qa` in **full mode** — this is a cross-cutting integration test, not a diff-aware check.

If QA finds issues:
- Identify which workstream introduced the issue
- Launch a targeted `/bugfix` agent to fix it
- Re-run QA after the fix

---

## Phase P8: Ship

If the user wants a single release:
- Read `~/.claude/skills/gstack/ship/SKILL.md` and execute `/ship`

If each workstream already shipped its own PR (the normal case):
- Report the full list of merged PRs
- Run `/retro` to summarize the full body of work

---

## Parallel Pipeline State

Track these additional values in parallel mode:

| Field | Set by | Used by |
|-------|--------|---------|
| `workstreams[]` | Phase P1 | All parallel phases |
| `waves[]` | Phase P2 | Phase P4 (launch order) |
| `agent_status{}` | Phase P4/P5 | Phase P5 (monitoring), P6 (integration) |
| `pr_urls{}` | Phase P5 | Phase P6 (merge order) |
| `merge_order[]` | Phase P6 | Phase P6 (sequential merge) |

---

## Pipeline State (single mode)

Track these values across phases (in conversation context — no external storage needed):

| Field | Set by | Used by |
|-------|--------|---------|
| `type` | Phase 1 (Triage) | Phase 2 (plan depth), Phase 6 (QA mode) |
| `scope` | Phase 1 (Triage) | Phase 2 (CEO review mode), Phase 6 (QA mode) |
| `plan_review_rounds` | Phase 3 | Phase 3 (escalation at 3) |
| `code_review_rounds` | Phase 5 | Phase 5 (escalation at 3) |

---

## Error Handling

| Situation | Action |
|-----------|--------|
| A gstack skill is not installed | Stop and tell user: "Required skill `/X` not found. Run `cd ~/.claude/skills/gstack && ./setup` to install." |
| User says "skip" during any phase | Skip that phase, continue to next |
| User says "abort" or "stop" | Stop the pipeline immediately |
| Phase gate fails 3 times | Ask user to intervene — do not loop forever |
| Merge conflict in Phase 7 | `/ship` handles this — do not duplicate |
| Parallel agent fails | Report failure with reason, ask user: Retry / Skip / Abort |
| Parallel agent times out | Report timeout, ask user to intervene |
| Merge conflict between workstreams | Report conflicting files, ask user to resolve or assign a `/bugfix` agent |
| Too many workstreams (>8) | Consolidate related workstreams until ≤8 — explain the merges to the user |

---

## Important Rules

1. **One command, full lifecycle.** The user says `/orchestrate "task"` and the pipeline runs end-to-end.
2. **Phase gates are mandatory.** Phase 3 (plan review) and Phase 5 (code review) are quality gates — do not skip them.
3. **Respect scope.** SCOPE REDUCTION means less work, not lower quality. SCOPE EXPANSION means more ambition, confirmed by the user.
4. **Do not duplicate skill logic.** Each phase calls the corresponding gstack skill. Do not reimplement what `/review`, `/qa`, or `/ship` already do.
5. **Escalate, don't loop.** Max 3 rejection rounds at any gate. After 3, ask the user.
6. **Progress updates.** At each phase transition, output a one-line status: `Phase N → Phase N+1: <reason>`
7. **Parallel agents are isolated.** Each agent runs in its own worktree — they cannot see each other's changes until merge.
8. **Wave ordering is strict.** Never launch Wave N+1 until all agents in Wave N have completed and their PRs are merged.
9. **Skill routing matters.** Use `/bugfix` for bugs, `/debug` for investigation, `/orchestrate` for features — don't send everything through the same pipeline.
10. **User confirms decomposition.** Never fan out agents without Phase P3 user approval — parallel work is expensive and hard to undo.
