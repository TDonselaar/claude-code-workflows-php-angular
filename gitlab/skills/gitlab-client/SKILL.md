---
name: gitlab-client
description: GitLab client conventions for agent-driven work-item and issue handling. Defines saved-view name, label vocabulary, comment-marker protocol, branch naming, MR body template, and maps each operation to the GitLab Duo MCP tool. Loaded by recipe-gitlab-workitem and recipe-gitlab-issue.
---

# GitLab Client (Agent Protocol)

This skill is the shared contract used by the GitLab recipes, the scope classifier, and the bug reproducer. It does **not** run anything on its own — it defines names and shapes.

## Configuration (project-specific)

| Key | Default | Notes |
|---|---|---|
| Saved view for backlog | `Claude backlog` | Must pre-filter to label `agent:ready` (configured in GitLab UI) |
| Default base branch | `upgrade` | Overridable per work item via label |
| Comment marker version | `claude-agent:v1` | Bump only on breaking protocol change |
| Branch prefix | `agent/` | Keep short; GitLab imposes a 244-byte ref-name limit |
| Commit trailer | `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>` | Matches Claude Code default |

## Label Vocabulary

Labels the human configures in GitLab (the MCP server does not expose a create-label/add-label API, so applying labels is a human or GitLab-automation concern):

| Label | Purpose |
|---|---|
| `agent:ready` | Work item is queued for agent pickup |
| `agent:claimed` | Optional. If a GitLab automation promotes the comment marker into a label, this is the label |
| `agent:autonomous` | Skip human gates (Gate 1 + Gate 2) for this work item |
| `agent:manual` | Pause between every stage (overrides `agent:autonomous`) |
| `scope:backend` | Laravel / backend work |
| `scope:frontend` | Angular frontend work |
| `scope:editor` | Flutter editor work |
| `base:main` | Override base branch to `main` |
| `base:live` | Override base branch to `live` |
| `type:bug` | Route via `recipe-gitlab-issue` (for issues; for work items this label alone still routes as a bug fix) |

Multiple `scope:*` labels → multi-scope routing via `recipe-gitlab-monorepo-build`.

## MCP Tool Mapping

All calls use the prefix `mcp__55347110-1326-4bad-953f-d8fd59b0c0b6__` (abbreviated below as `mcp__gitlab__`).

| Purpose | Tool |
|---|---|
| Discover work items | `mcp__gitlab__get_saved_view_work_items` (pass saved view name = `Claude backlog`) |
| Read a single work item | `mcp__gitlab__search` (by iid/title) then `mcp__gitlab__get_issue` or get via the saved-view result payload |
| Read comments on a work item | `mcp__gitlab__get_workitem_notes` |
| Post comment on a work item | `mcp__gitlab__create_workitem_note` |
| Read an issue (bug) | `mcp__gitlab__get_issue` |
| Create an issue (for spin-off work) | `mcp__gitlab__create_issue` |
| Read MR metadata | `mcp__gitlab__get_merge_request` |
| Read MR diffs / commits / pipelines | `mcp__gitlab__get_merge_request_diffs` / `get_merge_request_commits` / `get_merge_request_pipelines` |
| Create MR | `mcp__gitlab__create_merge_request` |
| Pipeline jobs / manage | `mcp__gitlab__get_pipeline_jobs` / `mcp__gitlab__manage_pipeline` |
| Semantic code search | `mcp__gitlab__semantic_code_search` |
| Keyword search | `mcp__gitlab__search` |
| List labels | `mcp__gitlab__search_labels` |
| MCP server version | `mcp__gitlab__get_mcp_server_version` (use on first run to sanity-check) |

**No add-label, no assign, no close-issue tool**. State transitions in GitLab that require those operations are done by the human reviewer or by a GitLab automation that reacts to the comment marker. Agents must not fail because they "can't add a label" — the comment marker is authoritative.

## Comment Marker Protocol

Every agent comment starts with one HTML comment header line, followed by a human-readable body:

```
<!-- claude-agent:v1 state=<state> stage=<stage> run=<uuid> branch=<branch> base=<base-branch> -->
🤖 **<Status>** — <body>
```

**Field definitions**:

| Field | Required | Example values |
|---|---|---|
| `state` | yes | `claimed`, `analyzing`, `awaiting_answer`, `designing`, `designed`, `planning`, `implementing`, `verifying`, `mr_opened`, `merged`, `blocked` |
| `stage` | no | `requirement-analysis`, `scope-classification`, `design`, `review`, `task-decomposition`, `execution:1/5`, etc. — freeform |
| `run` | yes | UUID generated at recipe entry; stable across comments within one recipe invocation |
| `branch` | conditional | present once branch is created |
| `base` | conditional | present once base branch is resolved |

**State machine**:

```
claimed
  ├─ analyzing
  │    ├─ awaiting_answer ─(human reply)─ analyzing
  │    └─ designing
  │         ├─ awaiting_answer ─(human reply)─ designing
  │         └─ designed
  │              └─ planning
  │                   └─ implementing
  │                        └─ verifying
  │                             └─ mr_opened
  │                                  └─ merged
  └─ blocked (terminal; requires human reclassification)
```

**Reading human input**:

- Notes **without** the `claude-agent:v1` header are human messages.
- On recipe re-entry, look at the last agent note: its `state` is the resume point.
- A human note posted **after** an `awaiting_answer` note is the reply; resume on next recipe invocation.
- A human note containing `@claude` is an explicit resume signal (resume even if last agent state was not `awaiting_answer`).

## Branch Naming

- Feature: `agent/wi-<iid>-<slug>` — work-item IID + kebab-cased title (truncate to 40 chars)
- Bug: `agent/issue-<iid>-<slug>`
- Sub-scope (multi-scope run): same branch; tasks routed per-file

Example: `agent/wi-4712-billing-coupon-validation`

## MR Body Template

```
## Summary
<1–3 bullets of what changed>

## Work item
Closes #<iid>

## Scope
- <scope labels used>

## Design
- docs/design/<feature>-design.md

## Acceptance Criteria
- [ ] <pulled from Design Doc AC>
- [ ] ...

## Test plan
- [ ] <stack-specific quality loop output summary>
- [ ] <per-scope manual verifications if any>

## Notes
<Any excluded checks, pre-merge platform builds, or follow-up work>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

Bug MRs add a **Root cause** section above **Test plan**:

```
## Root cause
<1 paragraph>

## Reproduction
- Test: <path to reproduction test>
- Before/after behaviour: <short contrast>
```

## Standard Comment Bodies

Recipes reuse these verbatim where applicable — consistency matters for human scan-ability.

### Claim

```
<!-- claude-agent:v1 state=claimed run=<uuid> -->
🤖 **Claimed** — picking up this work item for agent flow. Next step: scope classification.
```

### Awaiting answer (Gate 1 — scope)

```
<!-- claude-agent:v1 state=awaiting_answer stage=scope-classification run=<uuid> -->
🤖 **Awaiting answer** — I classified this as scope: **<classification>** with rationale below. Reply with `@claude confirm` to proceed, or correct the scope with `@claude scope: <laravel|angular|flutter|...>`.

Rationale:
- <reason 1>
- <reason 2>
```

### Awaiting answer (Gate 2 — design)

```
<!-- claude-agent:v1 state=awaiting_answer stage=design run=<uuid> branch=<branch> base=<base> -->
🤖 **Design ready** — Design Doc drafted at `<path>` on branch `<branch>` (base: `<base>`).

Summary:
<1–2 paragraphs covering key decisions>

Open questions (if any):
- <q1>

Reply `@claude approve` to proceed to planning + implementation, or post feedback and I'll revise.
```

### MR opened

```
<!-- claude-agent:v1 state=mr_opened run=<uuid> branch=<branch> -->
🤖 **MR opened** — !<mr-iid> — <mr url>

All tasks committed. Per-task quality loop: passed.
Excluded checks (manual pre-merge): <list or "none">
```

### Blocked

```
<!-- claude-agent:v1 state=blocked run=<uuid> -->
🤖 **Blocked** — <reason>

What I tried:
- <attempt>

What I need from you:
- <ask>
```

## Failure Modes

- **MCP server unreachable**: abort the recipe, print the error, do not partially proceed. No silent retries — the human picks up.
- **Comment post fails**: retry once. If still failing, escalate (text response to user); do not leave the work in a half-state.
- **MR creation fails** (conflicts on base, missing permissions): post a `blocked` comment with the specific error message and MR branch name so the human can inspect.

## First-Run Sanity Check

On the first recipe invocation in a session, call `mcp__gitlab__get_mcp_server_version`. If it fails, the MCP isn't connected — escalate with setup instructions instead of proceeding.
