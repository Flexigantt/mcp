---
name: flexigantt
description: Answer questions about a FlexiGantt project open in the app — why a date moved, how much float or deadline buffer is left, whether the forecast is stale — and export Gantt/Roadmap views, using the bundled MCP tools. Read-only. Use for inspecting or exporting an actual schedule via MCP, not narrative status-report authoring, hypothetical scenario comparison, or creating/importing/mutating a project.
---

# FlexiGantt (MCP)

FlexiGantt MCP Skill Version: 1
Requires a FlexiGantt build with Settings > MCP enabled (off by default) and
this agent connected to it — check whether `list_open_projects` is available
before relying on anything below.

These tools read a project already open in the FlexiGantt app, or bytes you
supply directly. They never create, import, or mutate a project — FlexiGantt
has no MCP write tools yet. For a current-state executive, milestone, or
HTML status report, use the `flexigantt-status-report` skill.

Do not parse or edit `.flg` files directly: they are SQLite-backed
application documents, and direct edits bypass validation and scheduling.

## Read the project

Call `list_open_projects` first to get a valid id, then:

| Tool | Returns |
|---|---|
| `list_open_projects` | Every project currently open in the app: `open_project_id`, `name`, `file_path`. |
| `get_project` | Identity: `id`, `name`, `start_date`, `task_count`, `resource_count`, `velocity_factor`. |
| `list_tasks` | Every task: WBS, dates, completion, status, float, resource, predecessors. |
| `get_task` | Full detail for one task, matched by UUID, WBS number, or unambiguous name (`task_id`). |
| `get_schedule_drivers` | Predecessors, dependency type/lag, total and free float, and the downstream deadline milestones a task drives (`task_id`). |
| `get_critical_paths` | The current driving sequences through the project. |
| `get_status` | Milestone and dependency-gate health: signal, forecast, baseline, variance, buffer, progress. |

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
Settings > MCP.

## Export views

Three tools render a project without touching disk:

| Tool | Produces |
|---|---|
| `export_gantt_pdf` | The full Gantt chart as a PDF. |
| `export_roadmap_pdf` | A paginated Roadmap PDF. |
| `export_roadmap_pptx` | A single-slide, editable Roadmap PowerPoint deck — native shapes, not a locked image. |

Each takes the same `open_project_id`/`project_data` argument the read
tools use. The result has two parts: a JSON envelope (`export`,
`mime_type`, `byte_size`) as the standard text content, and the rendered
file as an embedded binary resource (base64 in `resource.blob`) — decode
it and save it wherever the user wants; the MCP server itself never writes
the file anywhere.

## Response envelope

Every tool returns `schema_version`, `ok`, and `result` on success; on
failure, `ok: false` and an `error` with a stable `code` and human-readable
`message`. Preserve the error code; do not guess around a failure.

## Safety rules

- Never edit the `.flg` database with SQLite tools or scripts.
- These tools never mutate or save a project. There is no MCP path to
  create, import, or change a task — that requires the FlexiGantt app
  itself, or the experimental `flexigantt` CLI skill if it's installed.
  Do not attempt to work around this by writing to the `.flg` file.
- Exports never write to disk themselves — decoding and saving
  `resource.blob` is this agent's job.
