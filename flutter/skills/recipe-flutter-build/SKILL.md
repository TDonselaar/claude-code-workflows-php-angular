---
name: recipe-flutter-build
description: Execute Flutter task files autonomously. Routes *-flutter-task-*.md files through task-executor-flutter + quality-fixer-flutter. Use when Flutter work plan/tasks exist.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

**Execution Protocol**:
1. **Delegate all work through Agent tool**
2. **4-step task cycle**: task-executor-flutter → escalation check → quality-fixer-flutter → commit
3. **Enter autonomous mode** when invoked with existing task files
4. **Scope**: complete when all Flutter tasks are committed or escalation occurs

**CRITICAL**: Run `dev-workflows-flutter:quality-fixer-flutter` before every commit.

Work plan: $ARGUMENTS

## Pre-execution Prerequisites

```bash
! ls docs/plans/tasks/*-flutter-task-*.md 2>/dev/null || ls docs/plans/tasks/*.md 2>/dev/null || echo "No task files found"
! test -f .fvmrc && echo "FVM pinned" || echo "no .fvmrc — use system flutter"
```

| State | Next Action |
|---|---|
| `*-flutter-task-*.md` exist | Autonomous execution |
| `*.md` with no stack suffix | Treat as Flutter tasks (fallback) |
| Only work plan exists | Confirm with user → invoke `dev-workflows:task-decomposer` instructing filenames `{plan}-flutter-task-{n}.md` |
| Only Design Doc exists | Invoke `dev-workflows:work-planner` → then decomposer |
| Nothing | Report missing prerequisites, stop |

## Task Execution Cycle

1. **TaskCreate** — "Confirm skill constraints" first, "Verify skill fidelity" last
2. **Invoke** `dev-workflows-flutter:task-executor-flutter` with task file path
3. **Check response**:
   - `escalation_needed` / `blocked` → escalate
   - `requiresTestReview: true` → invoke `dev-workflows:integration-test-reviewer`
   - `readyForQualityCheck: true` → proceed
4. **Invoke** `dev-workflows-flutter:quality-fixer-flutter` with `task_file`
5. **Check response**:
   - `stub_detected` → back to step 2
   - `blocked` → escalate
   - `approved` → proceed
6. **Commit** — one commit per task.

## Sub-agent Constraint Suffix

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-flutter-build scope. Use orchestrator-provided rules only.
```

## Quality Scope (per-task)

Per-task quality loop runs:

- `fvm flutter analyze`
- `fvm flutter test` (unit + widget tests)
- `fvm dart format --set-exit-if-changed .`
- Code-gen check: if any `.g.dart` or `.freezed.dart` is stale given the source, regenerate via `fvm dart run build_runner build --delete-conflicting-outputs`

**Excluded from per-task loop** (require a manual step before merge):

- Platform-specific integration tests (Android/iOS/desktop/web/Tizen)
- `fvm flutter build <target>` for each platform
- Golden tests on specific devices
- Firebase Crashlytics smoke tests

The `quality-fixer-flutter` agent will note excluded-but-recommended checks in its output so the orchestrator can surface them on the MR.

## Post-Implementation Verification

Invoke in parallel:
- `dev-workflows:code-verifier` (Design Doc + `git diff --name-only <base>...HEAD`)
- `dev-workflows:security-reviewer` (Design Doc + files)

Failure → consolidation task → re-run executor + quality-fixer → re-verify.

## Completion Report

```
Flutter implementation phase completed.
- Tasks: <n> Flutter task(s) committed
- Quality: fvm flutter analyze / fvm flutter test / dart format — all passed
- Excluded (manual pre-merge): [list of platform builds / integration tests]
- Verifiers: code-verifier + security-reviewer — passed
```
