---
name: recipe-laravel-build
description: Execute Laravel task files autonomously. Routes *-laravel-task-*.md files through task-executor-laravel + quality-fixer-laravel. Use when Laravel work plan/tasks exist.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

**Execution Protocol**:
1. **Delegate all work through Agent tool** — invoke sub-agents, pass deliverable paths, report results
2. **Follow the 4-step task cycle exactly**: task-executor-laravel → escalation check → quality-fixer-laravel → commit
3. **Enter autonomous mode** when invoked with existing task files — that IS the batch approval
4. **Scope**: complete when all Laravel tasks are committed or an escalation occurs

**CRITICAL**: Run `dev-workflows-laravel:quality-fixer-laravel` before every commit.

Work plan: $ARGUMENTS

## Pre-execution Prerequisites

```bash
! ls docs/plans/tasks/*-laravel-task-*.md 2>/dev/null || ls docs/plans/tasks/*.md 2>/dev/null || echo "No task files found"
```

| State | Next Action |
|---|---|
| `*-laravel-task-*.md` exist | Autonomous execution |
| `*.md` with no stack suffix exist | Treat as Laravel tasks (fallback) |
| Only work plan exists | Confirm with user → invoke `dev-workflows:task-decomposer` with instruction: "name files `{plan}-laravel-task-{n}.md`" |
| Only Design Doc exists | Invoke `dev-workflows:work-planner` → then decomposer |
| Nothing | Report missing prerequisites, stop |

## Task Execution Cycle

For each `*-laravel-task-*.md` (or fallback) with uncompleted checkboxes:

1. **TaskCreate** — register work steps. First: "Confirm skill constraints". Last: "Verify skill fidelity".
2. **Invoke** `dev-workflows-laravel:task-executor-laravel` via Agent tool; pass task file path in prompt; receive structured response.
3. **Check response**:
   - `status: "escalation_needed"` or `"blocked"` → STOP, escalate to user
   - `requiresTestReview: true` → invoke `dev-workflows:integration-test-reviewer`; `needs_revision` returns to step 2, `approved` proceeds
   - `readyForQualityCheck: true` → proceed
4. **Invoke** `dev-workflows-laravel:quality-fixer-laravel` with `task_file` = the task path.
5. **Check response**:
   - `stub_detected` → return to step 2 with `incompleteImplementations[]`
   - `blocked` → STOP, escalate to user
   - `approved` → proceed
6. **Commit** — one commit per task. Message format: `<verb>: <task summary>` (imperative, lowercase verb, no trailing period). Include the task file path in the commit body.

## Sub-agent Constraint Suffix

Append to every sub-agent prompt:

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-laravel-build scope. Use orchestrator-provided rules only.
```

## Post-Implementation Verification (after all tasks)

Invoke in parallel:
- `dev-workflows:code-verifier` (`doc_type: design-doc`, Design Doc path, `code_paths` from `git diff --name-only <base>...HEAD`)
- `dev-workflows:security-reviewer` (Design Doc path + file list)

Consolidate results. On failure, produce a consolidation task → re-run executor + quality-fixer → re-verify only the failing reviewer. Repeat until pass or `blocked`.

## Completion Report

```
Laravel implementation phase completed.
- Tasks: <n> Laravel task(s) committed
- Quality: composer test / PHPStan / Laravel Pint — all passed
- Verifiers: code-verifier + security-reviewer — passed
```
