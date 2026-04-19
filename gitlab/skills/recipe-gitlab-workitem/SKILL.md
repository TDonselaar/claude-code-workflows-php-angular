---
name: recipe-gitlab-workitem
description: Pick up the next GitLab work item from the agent backlog, classify scope, design, plan, implement, and open an MR. Communicates progress via work-item comments using the claude-agent v1 marker protocol. Two human gates (scope and design) unless the work item is labelled `agent:autonomous`.
disable-model-invocation: true
---

## Orchestrator Definition

**Core Identity**: "I am an orchestrator." (see dev-workflows:subagents-orchestration-guide skill)

Loaded skill: `gitlab-client` — single source of truth for configuration, labels, tools, comment protocol, branch naming, MR template.

## Execution Protocol

1. Delegate work via Agent tool; use GitLab MCP tools (`mcp__55347110-*`) for all GitLab reads/writes
2. Honour two human gates by default: **Gate 1 — scope classification**, **Gate 2 — design approval**
3. Skip gates if the work item carries label `agent:autonomous`. Pause between every stage if it carries `agent:manual`
4. All agent comments use the comment-marker protocol defined in `gitlab-client`
5. Artifacts written inside the monorepo working directory: `docs/design/`, `docs/plans/`, `docs/plans/tasks/`

Target work item: `$ARGUMENTS` (optional work-item IID; if absent, the recipe picks from the saved-view backlog)

## Pre-execution Sanity Check

1. `mcp__gitlab__get_mcp_server_version` — if it fails, escalate with setup instructions and stop
2. Confirm the working directory is the monorepo root (the codebase we're acting against, not this workflow repo). If not, ask the user

## Phase 0 — Select Work Item

If `$ARGUMENTS` is a work-item IID:
- Fetch it directly
- Skip to Phase 1

Else:
- `mcp__gitlab__get_saved_view_work_items` with the saved view from `gitlab-client` (default `Claude backlog`)
- For each returned work item (oldest first):
  - `mcp__gitlab__get_workitem_notes` — look for a `claude-agent:v1` marker
  - Parse the last such marker's `state`:
    - `claimed` / `analyzing` / `designing` / `planning` / `implementing` / `verifying` — already in progress by another run; **skip** (do not collide)
    - `awaiting_answer` — check for a human note posted **after** it → **this one is eligible** (resume)
    - `mr_opened` / `merged` / `blocked` — terminal, skip
    - No marker at all → **eligible for claim**
- Pick the first eligible work item. If none, post a text response: "No eligible work items in the backlog." and stop

## Phase 1 — Claim (or Resume)

Determine whether this is a fresh claim or a resume:

**Fresh claim** (no prior marker, or only `claimed` from a prior aborted run without progress):
- Generate new `run` UUID
- Post the Claim comment (see `gitlab-client` → Standard Comment Bodies → Claim)

**Resume** (last marker was `awaiting_answer`, human replied):
- Reuse the `run` UUID from the awaiting marker
- Do not post another claim; just proceed to the resume point indicated by the awaiting marker's `stage`

## Phase 2 — Scope Classification

Invoke `dev-workflows-gitlab:scope-classifier` via Agent tool:
- subagent_type: `dev-workflows-gitlab:scope-classifier`
- prompt: work item title + description, list of available scope labels, path to the monorepo root so the agent can read code

Agent returns a classification (`laravel` / `angular` / `flutter` / any combination) with rationale.

### Gate 1 — Scope Approval

If work item has label `agent:autonomous` → **skip** Gate 1, proceed to Phase 3.

Else:
- Post `awaiting_answer` comment (see `gitlab-client` → Awaiting answer (Gate 1 — scope))
- **Exit recipe** cleanly. The user's `@claude` reply triggers the next recipe run, which resumes here.

On resume:
- Read latest human notes after the awaiting marker
- If contains `@claude confirm` or equivalent → adopt the proposed classification
- If contains `@claude scope: <...>` → adopt the human's override
- If ambiguous → post a clarifying `awaiting_answer` and exit again

## Phase 3 — Resolve Base Branch

Order of precedence:
1. Label `base:main` → `main`
2. Label `base:live` → `live`
3. Default from `gitlab-client` → `upgrade`

## Phase 4 — Branch + Requirement Analysis

Create branch `agent/wi-<iid>-<slug>` off the resolved base (actual Git creation happens during Phase 5/6 when code is being modified — Phase 4 only plans the name).

Invoke `dev-workflows:requirement-analyzer` via Agent tool with the work-item description and the chosen scope(s). Result: scale determination (Small / Medium / Large) + technical requirements summary.

Post a short analysis comment:

```
<!-- claude-agent:v1 state=analyzing stage=requirement-analysis run=<uuid> -->
🤖 **Analyzing** — scale: <Small|Medium|Large>. Next: <prd-creation | codebase-analysis | direct-design>.
```

## Phase 5 — Design

Route by scale:

- **Large**: `dev-workflows:prd-creator` → `dev-workflows:codebase-analyzer` → stack-specific `technical-designer-<stack>`
- **Medium**: `dev-workflows:codebase-analyzer` → stack-specific `technical-designer-<stack>`
- **Small**: stack-specific `technical-designer-<stack>` directly

Then: `dev-workflows:code-verifier` + `dev-workflows:document-reviewer` + `dev-workflows:design-sync` (for multi-scope).

Design Doc path: `docs/design/<feature>-design.md` inside the monorepo.

For multi-scope (>1 `scope:*` label):
- Invoke one `technical-designer-<stack>` per stack in parallel, then run `dev-workflows:design-sync` to reconcile cross-layer contracts

### Gate 2 — Design Approval

If work item has label `agent:autonomous` → **skip** Gate 2, proceed to Phase 6.

Else:
- Post `awaiting_answer` (Gate 2) comment with Design Doc path + summary
- **Exit recipe**. User `@claude approve` reply triggers resume.

On resume:
- Read human feedback after the Gate 2 marker
- If `@claude approve` → proceed
- If human posted revisions → re-invoke the designer with the feedback; re-post Gate 2 and exit again

## Phase 6 — Planning + Decomposition

Invoke in sequence:
1. `dev-workflows:work-planner` → writes `docs/plans/<feature>-plan.md`
2. `dev-workflows:task-decomposer` → writes `docs/plans/tasks/<feature>-<stack>-task-<n>.md`
   - Single-scope: all tasks share one stack suffix
   - Multi-scope: each task's suffix matches the file paths it touches

Post a planning comment:

```
<!-- claude-agent:v1 state=planning stage=task-decomposition run=<uuid> -->
🤖 **Planning** — <n> task(s) decomposed:
- <task-1.md>
- <task-2.md>
...
```

## Phase 7 — Implementation

Route by scope:

- Single-scope: invoke `dev-workflows-<stack>:recipe-<stack>-build` (Laravel / Angular / Flutter)
- Multi-scope: invoke `dev-workflows-gitlab:recipe-gitlab-monorepo-build`

Under the hood, those recipes run the 4-step cycle (executor → escalation check → quality-fixer → commit) per task.

During implementation, post **consolidated progress comments** — one per stack, updated at major milestones rather than every task:

```
<!-- claude-agent:v1 state=implementing stage=execution:3/5 run=<uuid> branch=<branch> base=<base> -->
🤖 **Implementing** — Laravel: 2/3 tasks committed · Angular: 1/2 tasks committed. Continuing.
```

(Comments cannot be updated via the MCP — post at three breakpoints per stack: started, halfway, complete.)

## Phase 8 — Verification

Post:

```
<!-- claude-agent:v1 state=verifying run=<uuid> -->
🤖 **Verifying** — running code-verifier + security-reviewer.
```

Invoke in parallel (per stack involved):
- `dev-workflows:code-verifier`
- `dev-workflows:security-reviewer`

If any returns fail → consolidate findings into one task file, run the appropriate stack's executor + quality-fixer, re-verify only the failing reviewer. Repeat until pass or `blocked` → escalate.

## Phase 9 — Open MR

`mcp__gitlab__create_merge_request`:
- source: branch
- target: resolved base branch
- title: `<work-item title>` (truncate to 70 chars)
- body: MR body template from `gitlab-client` (filled with summary, acceptance criteria from Design Doc, test plan from quality-fixer output, excluded checks from Flutter recipe if applicable)

Post the MR-opened comment with the MR IID and URL.

## Phase 10 — End

Return a short text summary to the user: work item iid, scope(s), tasks committed, MR link. Stop.

## Blocked / Error Paths

- Any phase can transition to `blocked` if a sub-agent returns `blocked` with a specification question — post a `blocked` comment quoting the sub-agent's reason and stop. No destructive actions on block.
- If an MCP call fails transiently, retry once. If it still fails, post `blocked` with the specific error and stop.

## Sub-agent Constraint Suffix

Append to every sub-agent prompt:

```
[SYSTEM CONSTRAINT]
This agent operates within recipe-gitlab-workitem scope. Use orchestrator-provided rules only.
```

## Forbidden

- Merging the MR automatically
- Closing the work item automatically (MCP has no close-issue tool anyway)
- Force-pushing the agent branch
- Writing to directories outside the monorepo
- Posting comments without the `claude-agent:v1` marker (breaks resume logic)
