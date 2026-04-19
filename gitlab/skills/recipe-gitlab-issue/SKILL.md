---
name: recipe-gitlab-issue
description: Pick up a GitLab issue (bug), reproduce it with a failing test, diagnose root cause, apply the minimal fix, open an MR. Uses the claude-agent v1 marker protocol for progress comments. Differs from recipe-gitlab-workitem by reproducing first and skipping full technical design.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

Loaded skill: `gitlab-client`.

## When to Use

Route here (not to `recipe-gitlab-workitem`) when the issue is a bug:
- Label `type:bug`
- Or title prefix `[Bug]`
- Or the reporter's description describes current-vs-expected behaviour

Otherwise, the work goes through `recipe-gitlab-workitem` (feature/task flow).

## Execution Protocol

1. Delegate via Agent tool; use GitLab MCP (`mcp__55347110-*`) for GitLab reads/writes
2. **Reproduction-first**: before any fix design, write a failing test that demonstrates the bug
3. Minimal-fix policy — the smallest code change that makes the failing test pass without breaking others; no incidental refactors
4. Use the marker protocol from `gitlab-client`

Target issue: `$ARGUMENTS` (issue IID; required — this recipe does not scan for issues autonomously)

## Pre-execution Sanity Check

1. `mcp__gitlab__get_mcp_server_version` — if it fails, escalate and stop
2. Confirm working directory is the monorepo root

## Phase 0 — Read Issue

- `mcp__gitlab__get_issue` with IID
- `mcp__gitlab__get_workitem_notes` (issues share the notes API)
- Parse the last `claude-agent:v1` marker:
  - No marker → fresh claim
  - `awaiting_answer` with subsequent human reply → resume
  - Any in-progress state from another run → text-response to user, stop (collision)
  - Terminal (`mr_opened` / `merged` / `blocked`) → text-response, stop

## Phase 1 — Claim

Generate `run` UUID. Post:

```
<!-- claude-agent:v1 state=claimed run=<uuid> -->
🤖 **Claimed** — bug fix flow starting. Next step: reproduction.
```

## Phase 2 — Scope Classification

Invoke `dev-workflows-gitlab:scope-classifier` with the issue title + description + referenced file paths (extract paths with a regex over the description; pass as hints).

For bugs, scope classification runs **without** Gate 1 by default — the fix path is narrow and the classifier's confidence is usually high. However: if the classifier returns `confidence: low` OR multiple scopes, treat it as a Gate 1 pause (same pattern as `recipe-gitlab-workitem` Phase 2).

## Phase 3 — Resolve Base Branch

Same precedence as workitem recipe: `base:main` → `base:live` → default `upgrade`.

## Phase 4 — Reproduction

Invoke `dev-workflows-gitlab:bug-reproducer` via Agent tool:
- subagent_type: `dev-workflows-gitlab:bug-reproducer`
- prompt: issue title, description, any logs/stack traces, classified scope(s)

Agent returns one of:

- `status: "reproduced"` — a failing test committed on a new branch `agent/issue-<iid>-<slug>`, with test path and failure output
- `status: "cannot_reproduce"` — with a list of information needed from the reporter

If `cannot_reproduce`:
- Post `awaiting_answer` comment with the specific asks
- Exit recipe. Human reply triggers resume.

If `reproduced`:
- Post:

```
<!-- claude-agent:v1 state=analyzing stage=reproduction run=<uuid> branch=<branch> base=<base> -->
🤖 **Reproduced** — failing test at `<test-path>` on branch `agent/issue-<iid>-<slug>`. Diagnosing root cause next.
```

## Phase 5 — Root Cause

Invoke `dev-workflows:investigator` with the failing test, the reproduction branch, and the classified scope(s). Result: root cause summary + suggested minimal fix location.

Post:

```
<!-- claude-agent:v1 state=designing stage=root-cause run=<uuid> branch=<branch> base=<base> -->
🤖 **Diagnosing** — root cause: <1–2 sentences>. Minimal fix planned at `<file:line>`.
```

## Phase 6 — Minimal Fix Plan

**Skip the full technical-designer flow.** Invoke `dev-workflows:task-decomposer` directly with:
- Input: investigator's root-cause summary + suggested fix location
- Output constraint: exactly **one** task file, named `docs/plans/tasks/<issue>-fix-<stack>-task-1.md`, containing:
  - The reproduction test reference
  - The minimal fix description
  - A **mandatory regression test** checkbox (the original failing test becomes the regression test — it was written in Phase 4)

Post:

```
<!-- claude-agent:v1 state=planning run=<uuid> -->
🤖 **Fix planned** — 1 task: `<task file path>`.
```

## Phase 7 — Execution

Route by scope (single scope expected; multi-scope triggers escalation to a human since bug fixes rarely span stacks — escalate with a `blocked` comment if multi-scope):

- Laravel → `dev-workflows-laravel:task-executor-laravel` → `dev-workflows-laravel:quality-fixer-laravel` → commit
- Angular → `dev-workflows-angular:task-executor-angular` → `dev-workflows-angular:quality-fixer-angular` → commit
- Flutter → `dev-workflows-flutter:task-executor-flutter` → `dev-workflows-flutter:quality-fixer-flutter` → commit

On executor `escalation_needed` → escalate to user; do not force a fix.

Post:

```
<!-- claude-agent:v1 state=implementing run=<uuid> branch=<branch> base=<base> -->
🤖 **Fix applied** — regression test passes. Running review.
```

## Phase 8 — Review

Invoke in parallel:
- `dev-workflows:code-reviewer`
- `dev-workflows:security-reviewer`

If either fails → consolidate into a follow-up micro-task → executor → quality-fixer → re-review. Do not expand scope.

## Phase 9 — Open MR

`mcp__gitlab__create_merge_request` with the bug MR body template (includes Root cause + Reproduction sections — see `gitlab-client`).

Title format: `[Bug] <short description>` (under 70 chars).

Body **Closes #<iid>** reference so GitLab links on merge.

Post the MR-opened comment.

## Phase 10 — End

Text summary to the user: issue iid, root cause one-liner, MR link.

## Cannot-Reproduce Flow

When `bug-reproducer` returns `cannot_reproduce`:

```
<!-- claude-agent:v1 state=awaiting_answer stage=reproduction run=<uuid> -->
🤖 **Cannot reproduce** — I wasn't able to reproduce the bug as described. To proceed I need:

- <specific ask 1>
- <specific ask 2>

Reply with additional context and `@claude` to resume. Alternatively, if this is intermittent or environment-specific, remove label `agent:ready` so the issue routes to a human.
```

## Sub-agent Constraint Suffix

Append to every sub-agent prompt:

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-gitlab-issue scope. Use orchestrator-provided rules only.
```

## Forbidden

- Expanding the fix scope beyond the minimal change that makes the regression test pass (file unrelated refactors as a separate work item if the cleanup is needed)
- Skipping the reproduction test
- Merging the MR automatically
- "Fixing" the bug by changing the failing test
- Force-pushing the agent branch
