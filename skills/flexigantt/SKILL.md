---
name: flexigantt
description: Answer questions about a FlexiGantt project open in the app — why a date moved, how much float or deadline buffer is left, whether the forecast is stale — explain how FlexiGantt behaves from its User Guide, and export Gantt/Roadmap views, using the bundled MCP tools. Use for inspecting or exporting an actual schedule via MCP, not narrative status-report authoring (flexigantt-status-report), creating or importing a project (flexigantt-create), hypothetical scenario comparison, or changing an existing project.
---

# FlexiGantt (MCP)

FlexiGantt MCP Skill Version: 2
Requires FlexiGantt 1.15 or later with Settings > MCP Server enabled (off
by default) and this agent connected to it — check whether
`list_open_projects` is available before relying on anything below.

These tools read a project already open in the FlexiGantt app, or bytes you
supply directly, and never change it. To build a new project from a
description, a CSV or Excel file, or MS Project XML, use the
`flexigantt-create` skill. No MCP tool edits an existing project. For a
current-state executive, milestone, or HTML status report, use the
`flexigantt-status-report` skill. For "what's
the plan for this week" or "which milestones should I focus on," use the
`flexigantt-weekly-plan` skill instead — it covers `get_weekly_plan` and
`get_critical_milestones`, not listed in the table below.

Do not parse or edit `.flg` files directly: they are SQLite-backed
application documents, and direct edits bypass validation and scheduling.

## Read the project

Call `list_open_projects` first to get a valid id, then:

| Tool | Returns |
|---|---|
| `list_open_projects` | Every project currently open in the app: `open_project_id`, `name`, `file_path`. |
| `get_project` | Identity: `id`, `name`, `start_date`, `task_count`, `resource_count`. |
| `list_tasks` | Every task: WBS, dates, completion, status, float, resource, predecessors, `notes` (Markdown task notes — acceptance criteria, guidance, links; null when none), and `frozen_velocity` (the velocity a started task keeps; null until it starts). |
| `get_task` | Full detail for one task, matched by UUID, WBS number, or unambiguous name (`task_id`). |
| `get_schedule_drivers` | Predecessors, dependency type/lag, total and free float, and the downstream deadline milestones a task drives (`task_id`). |
| `get_critical_paths` | The current driving sequences through the project. |
| `get_status` | Milestone and dependency-gate status: `schedule_status`, `health`, `stale`, forecast, baseline, variance (`_d`), buffers (`_wd`, unit in `buffer_unit`), progress. |

Every tool except `list_open_projects` takes exactly one of:

- `open_project_id` — an id from `list_open_projects`. Reads the live,
  in-memory state of that open document, including unsaved edits.
- `project_data` — base64-encoded `.flg` file contents, for a project the
  app doesn't have open as a document window. The app process itself still
  needs to be running with MCP enabled — this only removes the need for
  that specific project to be open in a window.

For repeated reads across a session, the full `list_tasks` and `get_status`
payloads are also available as MCP resources, `flexigantt://<open-project-id>/tasks`
and `.../status` — fetch these once instead of re-calling the matching tool
on every turn.

If these tools are not available in this session, the user has not enabled
the MCP server or connected this agent to it: point them to FlexiGantt >
Settings > MCP Server.

## Explain how FlexiGantt works

`get_help` reads the FlexiGantt User Guide: scheduling, progress, links,
milestones, views, shortcuts. Call it without arguments to list the
chapters, with `chapter` (an id from that list) to read one, or with
`query` to get the passages that mention some words. Use it when the user
asks why the app behaves as it does (how effort becomes duration, what a
dependency gate is, how health is judged) instead of answering from
memory, and quote the guide's wording where it matters.

## Export views

Three tools render a project without touching disk:

| Tool | Produces |
|---|---|
| `export_gantt_pdf` | The Gantt chart as File → Export → PDF draws it: task table, timeline, bars, dependency lines, every row expanded. |
| `export_roadmap_pdf` | The Roadmap's slides as a PDF, one page per slide. |
| `export_roadmap_pptx` | The Roadmap's slides as an editable PowerPoint deck, one slide per page — native shapes, not locked images. |

Each takes the same `open_project_id`/`project_data` argument the read
tools use. The result has two parts: a JSON envelope (`export`,
`mime_type`, `byte_size`) as the standard text content, and the rendered
file as an embedded binary resource (base64 in `resource.blob`) — decode
it and save it wherever the user wants; the MCP server itself never writes
the file anywhere.

Exports use the open project's window settings (or the app's defaults for
`project_data`). Override any of them per call; the user's window is never
changed:

- Gantt: `time_scale` (`days`/`weeks`/`quarters`), `show_dependencies`,
  `show_float`.
- Roadmap (PDF and PPTX): `layout` (`16:9`/`4:3`), `slide` (1-based; omit
  for every slide), `tasks` (show exactly these items, by UUID, WBS number
  or unique name), `color_scheme`, `time_scale`, and the inspector's
  switches `align_to_today`, `show_progress`, `show_deadline_buffer`,
  `show_float`, `show_task_column`, `show_status`, `alternate_row_bands`.
  The envelope's `slide_count` says how many slides there are, so you can
  export all of them, then one again by number.

Short bars carry no label on Roadmap slides; pass `show_task_column: true`
when every item must be named.

## Response envelope

Every tool returns `schema_version`, `ok`, and `result` on success; on
failure, `ok: false` and an `error` with a stable `code` and human-readable
`message`. Preserve the error code; do not guess around a failure.

## Safety rules

- Never edit the `.flg` database with SQLite tools or scripts.
- These tools never mutate or save a project. No MCP tool changes a task
  in an existing project — that requires the FlexiGantt app itself, or the
  experimental `flexigantt` CLI skill if it's installed. New projects come
  only from the `flexigantt-create` skill's tools. Do not work around this
  by writing to the `.flg` file.
- Exports never write to disk themselves — decoding and saving
  `resource.blob` is this agent's job.
