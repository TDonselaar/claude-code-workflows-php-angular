---
name: recipe-angular-build
description: Execute Angular task files autonomously. Routes *-angular-task-*.md files through task-executor-angular + quality-fixer-angular. Use when Angular work plan/tasks exist.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

**Execution Protocol**:
1. **Delegate all work through Agent tool**
2. **4-step task cycle**: task-executor-angular → escalation check → quality-fixer-angular → commit
3. **Enter autonomous mode** when invoked with existing task files
4. **Scope**: complete when all Angular tasks are committed or escalation occurs

**CRITICAL**: Run `dev-workflows-angular:quality-fixer-angular` before every commit.

Work plan: $ARGUMENTS

## Pre-execution Prerequisites

```bash
! ls docs/plans/tasks/*-angular-task-*.md 2>/dev/null || ls docs/plans/tasks/*.md 2>/dev/null || echo "No task files found"
```

| State | Next Action |
|---|---|
| `*-angular-task-*.md` exist | Autonomous execution |
| `*.md` with no stack suffix | Treat as Angular tasks (fallback) |
| Only work plan exists | Confirm with user → invoke `dev-workflows:task-decomposer` instructing filenames `{plan}-angular-task-{n}.md` |
| Only Design Doc exists | Invoke `dev-workflows:work-planner` → then decomposer |
| Nothing | Report missing prerequisites, stop |

## Task Execution Cycle

1. **TaskCreate** — first "Confirm skill constraints", last "Verify skill fidelity"
2. **Invoke** `dev-workflows-angular:task-executor-angular` with the task file path
3. **Check response**:
   - `escalation_needed` / `blocked` → escalate
   - `requiresTestReview: true` → invoke `dev-workflows:integration-test-reviewer`
   - `readyForQualityCheck: true` → proceed
4. **Invoke** `dev-workflows-angular:quality-fixer-angular` with `task_file`
5. **Check response**:
   - `stub_detected` → back to step 2
   - `blocked` → escalate
   - `approved` → proceed
6. **Commit** — one commit per task, imperative lowercase verb.

## Sub-agent Constraint Suffix

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-angular-build scope. Use orchestrator-provided rules only.
```

## Post-Implementation Verification

Invoke in parallel:
- `dev-workflows:code-verifier` (Design Doc + `git diff --name-only <base>...HEAD`)
- `dev-workflows:security-reviewer` (Design Doc + files)

Failure → consolidation task → re-run executor + quality-fixer → re-verify.

## Build-Path Invariant

Angular production output goes to `../editor/web`. A task that touches `angular.json` must preserve `outputPath`. The quality-fixer verifies this on any `angular.json` diff.

## Completion Report

```
Angular implementation phase completed.
- Tasks: <n> Angular task(s) committed
- Quality: ng test / ng lint / ng build (prod) — all passed
- Verifiers: code-verifier + security-reviewer — passed
```
