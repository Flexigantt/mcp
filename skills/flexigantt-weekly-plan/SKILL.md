---
name: flexigantt-weekly-plan
description: Answer "what's the plan for this week?" and "which milestones should I focus on?" for a FlexiGantt project open in the app, using its Iterations view and milestone/dependency-gate buffer, always checked against schedule staleness. Use for the current iteration's plan or a ranked milestone focus list — not for a full narrative status report (use flexigantt-status-report) or general schedule inspection (use flexigantt).
---

# FlexiGantt Weekly Plan & Critical Milestones (MCP)

FlexiGantt Weekly Plan MCP Skill Version: 1
Requires a FlexiGantt build with Settings > MCP Server enabled and this agent
connected to it — check whether `list_open_projects` is available before
relying on anything below.

Use this skill for exactly two questions: "what's the plan for this week?"
and "what are the critical milestones I should focus on?" Both are answered
from the project's Iterations view (its current-iteration cadence) and its
milestone/dependency-gate buffer tracking — never invent a "week" from raw
task dates when these tools are available.

For a full narrative status report (executive summary, exports, HTML), use
`flexigantt-status-report` instead. For general schedule inspection (why a
date moved, one task's float, exporting a view), use `flexigantt`.

## Read the plan

Call `list_open_projects` to get the project's `open_project_id`, then:

| Tool | Answers | Returns |
|---|---|---|
| `get_weekly_plan` | "What's the plan for this week?" | The current iteration's window, its tasks grouped into the categories the Iterations view uses (`must_start`, `must_finish`, `has_progress` flags on each), its planned milestones, and a `tasks_needing_update` list. |
| `get_critical_milestones` | "Which milestones should I focus on?" | Every milestone and dependency gate, ranked by least `remaining_buffer_wd` first then soonest `planned_finish`, each with its `schedule_status`, `health`, `stale` flag, buffer figures, and `blocking_stale_task_ids`. |

Both take the same `open_project_id`/`project_data` argument every other
FlexiGantt read tool takes, and read the live in-memory document.

## Never state a date or buffer without its staleness caveat

This is the one rule that matters most in this skill. A stale task is one
whose actual dates/progress haven't been recorded even though its planned
dates have already passed — `get_status`'s `stale_task_ids` concept, applied
here to the current iteration and to milestones specifically:

- `get_weekly_plan`'s response has a top-level `tasks_needing_update` array
  — every task in the current iteration that is stale. **Always lead with
  this list, not bury it**, before describing the rest of the plan: "Before
  trusting this plan, update: <names>." Each task entry inside `categories`
  also carries its own `is_stale` flag if you need per-task granularity.
- `get_critical_milestones`'s response has `stale_task_count` (project-wide)
  and, per milestone, `blocking_stale_task_ids` — the specific upstream
  tasks whose missing updates could be undermining that milestone's buffer
  number. When a milestone has a nonempty `blocking_stale_task_ids`,
  say so explicitly next to its buffer figure ("3 days remaining buffer,
  but this depends on 2 tasks that haven't been updated — treat this
  number as provisional until they are").
- If `get_critical_milestones` reports a milestone's own `stale` as
  `true`, its dates and buffer are not to be trusted at all — present it
  as needing attention for that reason (`stale_reason`: `past_due`, or
  `upstream` with the first stale task in `stale_upstream_id`), not by its
  (possibly stale) buffer number. `stale` is a separate flag: still report
  `schedule_status` and `health` exactly as returned, never renamed or
  softened.

Resolve `blocking_stale_task_ids`/`tasks_needing_update` ids to names using
the data already in the same tool's response (`get_weekly_plan`'s task
entries already carry `name`; for `get_critical_milestones`, cross-reference
`list_tasks` if you need names for ids that aren't themselves in the
`milestones` array). Do not expose raw UUIDs to the user.

## Interpret the responses faithfully

- `get_weekly_plan`'s `categories` are already in the order the Iterations
  view itself renders them; preserve that order and each category's `label`
  verbatim rather than inventing your own phrasing for "must start" vs
  "must finish" vs "has progress."
- `must_start`/`must_finish`/`has_progress` on a task entry are the
  Iterations view's own status-view groups — use them directly to
  answer "what must start/finish this week," don't re-derive them from
  dates yourself.
- `get_critical_milestones`'s ordering (least buffer first, then soonest)
  is already the "critical, ranked" list the question asks for — present it
  in the order returned rather than re-sorting by a different field unless
  the user asks for a different cut.
- A milestone's `wbs` field is its display id (the milestone id) —
  show it alongside the name for milestones, the same way FlexiGantt's own
  UI does; it is not present on task entries.
- Buffer keys end in `_wd` and variance in `_d`; read the buffer unit from
  `buffer_unit` — `wd` (working days), or `d` for a milestone scheduled on
  calendar days, in which case the `_wd` keys hold calendar days.
- `remaining_buffer_wd` and other buffer/variance fields may be `null`
  ("not applicable," e.g. no deadline or no critical-path finish date yet)
  — display "Not set" or omit, never convert `null` to `0`.

## Handle the "not configured yet" cases explicitly

`get_weekly_plan` returns a structured error, not a partial plan, in these
cases — surface the error's `message` directly to the user rather than
guessing at a plan:

- `ITERATION_PLANNING_NOT_CONFIGURED` — no iteration start date set. Point
  the user to Project Properties.
- `NO_ACTIVE_BASELINE` — no baseline exists/is active. Point the user to
  creating and activating one.
- `MISSING_BASELINE_DATES` — the active baseline is missing a start/finish
  entry for some in-scope task; the message names it. This is FlexiGantt's
  own "identify the offending task, don't guess a partial answer" rule —
  don't try to work around it by estimating that task's dates yourself.

`get_critical_milestones` has no such preconditions — an empty
`milestones` array simply means the project has no milestones or
dependency gates yet.

## Response envelope

Every tool returns `schema_version`, `ok`, and `result` on success; on
failure, `ok: false` and an `error` with a stable `code` and human-readable
`message`. Preserve the error code; do not guess around a failure.

## Safety rules

- Read-only, same as every other FlexiGantt MCP tool: these two never
  mutate or save a project.
- Never edit the `.flg` database with SQLite tools or scripts.
- If these tools are not available in this session, the user has not
  enabled the MCP server or connected this agent to it: point them to
  FlexiGantt > Settings > MCP Server.
