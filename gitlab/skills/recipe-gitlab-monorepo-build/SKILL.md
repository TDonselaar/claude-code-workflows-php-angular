---
name: recipe-gitlab-monorepo-build
description: Execute multi-scope task files across Laravel, Angular, and Flutter. Extends recipe-fullstack-build routing with per-stack suffixes. Use when a work item touches more than one of backend/frontend/editor.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

Extends the multi-layer pattern in [dev-workflows:recipe-fullstack-build](../../../backend/skills/recipe-fullstack-build/SKILL.md) with three stack-specific suffixes.

## Execution Protocol

1. Delegate all work via Agent tool
2. Route per task file suffix (see routing table below)
3. 4-step cycle per task: executor → escalation check → quality-fixer → commit
4. Scope complete when all tasks are committed or escalation occurs

**CRITICAL**: run the layer-appropriate quality-fixer before every commit.

Work plan: $ARGUMENTS

## Routing Table

| Filename pattern | Executor | Quality fixer |
|---|---|---|
| `*-laravel-task-*.md` | `dev-workflows-laravel:task-executor-laravel` | `dev-workflows-laravel:quality-fixer-laravel` |
| `*-angular-task-*.md` | `dev-workflows-angular:task-executor-angular` | `dev-workflows-angular:quality-fixer-angular` |
| `*-flutter-task-*.md` | `dev-workflows-flutter:task-executor-flutter` | `dev-workflows-flutter:quality-fixer-flutter` |
| `*-backend-task-*.md` | `dev-workflows:task-executor` | `dev-workflows:quality-fixer` |
| `*-frontend-task-*.md` | `dev-workflows-frontend:task-executor-frontend` | `dev-workflows-frontend:quality-fixer-frontend` |
| `*-task-*.md` (no layer prefix) | `dev-workflows:task-executor` | `dev-workflows:quality-fixer` (default) |

The React-specific rows exist so this recipe is drop-in compatible with the existing `dev-workflows-frontend` plugin for projects that use it. For a Laravel + Angular + Flutter monorepo, expect the first three rows only.

## Pre-execution Prerequisites

```bash
! ls docs/plans/tasks/*-task-*.md 2>/dev/null || echo "No task files found"
```

| State | Next Action |
|---|---|
| Tasks exist | Autonomous execution |
| Work plan exists, no tasks | Confirm with user → invoke `dev-workflows:task-decomposer` with layer-aware naming instruction: "use suffix `-laravel-task-n`, `-angular-task-n`, `-flutter-task-n` based on target file paths (`app/` → laravel, `src/app/` → angular, `lib/` → flutter)" |
| Design Doc(s) exist, no plan | `dev-workflows:work-planner` → decomposer |
| Nothing | Report missing prerequisites, stop |

## Task Execution Order

1. Determine dependency ordering within a scope (e.g. Laravel entity task before Laravel service task that uses it)
2. For cross-scope dependencies (e.g. Laravel API endpoint before Angular consumer), honour the order declared in the work plan. If the work plan doesn't declare it, default ordering is: Laravel → Angular → Flutter (backend-first)
3. A task failing halts execution for its scope but **not** necessarily for independent other scopes; however if any escalation is `design_compliance_violation`, halt the whole recipe and escalate to the user

## 4-Step Cycle (per task)

For each task file with uncompleted checkboxes:

1. **TaskCreate** — "Confirm skill constraints" first, "Verify skill fidelity" last
2. **Invoke** the executor per the routing table with the task file path
3. **Check response**:
   - `escalation_needed` / `blocked` → halt cycle, escalate
   - `requiresTestReview: true` → invoke `dev-workflows:integration-test-reviewer`
   - `readyForQualityCheck: true` → proceed
4. **Invoke** the stack-appropriate quality-fixer with `task_file`
5. **Check response**:
   - `stub_detected` → return to step 2 with `incompleteImplementations[]`
   - `blocked` → escalate
   - `approved` → proceed
6. **Commit** — one commit per task, imperative lowercase verb, commit body includes the task file path

## Sub-agent Constraint Suffix

Append to every sub-agent prompt:

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-gitlab-monorepo-build scope. Use orchestrator-provided rules only.
```

## Post-Implementation Verification

Invoke in parallel **per stack involved**:
- `dev-workflows:code-verifier` (once per Design Doc)
- `dev-workflows:security-reviewer` (full changed-files list)

Consolidate. On failure, consolidation task → re-run executor + quality-fixer for the affected scope only → re-verify.

## Completion Report

```
Monorepo implementation phase completed.
- Laravel: <n> tasks committed
- Angular: <n> tasks committed
- Flutter: <n> tasks committed
- Quality: per-stack loops all passed
- Verifiers: code-verifier + security-reviewer — passed
- Excluded (manual pre-merge): <list from Flutter recipe if applicable>
```
