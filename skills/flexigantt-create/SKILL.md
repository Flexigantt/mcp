---
name: flexigantt-create
description: Create a new FlexiGantt project via the bundled MCP tools — draft a plan from a description (tasks, hierarchy, effort, dependencies, milestones, deadlines, resources, notes), or import one from a CSV, Excel workbook, or MS Project XML file — and open it in the app or hand back the .flg file. Use when the user wants a new plan built or a file brought into FlexiGantt; not for changing an existing project (no MCP tool does that), reading a live schedule (use flexigantt), or status reports (use flexigantt-status-report).
---

# FlexiGantt: create and import projects (MCP)

FlexiGantt Create MCP Skill Version: 1
Requires FlexiGantt 1.15 or later with Settings > MCP Server enabled and
this agent connected to it — check whether `create_project` is available
before relying on anything below.

These tools build a **brand-new** project. They never change a project that
already exists: there is no MCP tool to edit, add or delete tasks in an open
plan. To change a plan, the user edits it in the app.

Do not write `.flg` files yourself: they are SQLite-backed application
documents, and only these tools produce a valid, scheduled one.

## Pick the tool

| Tool | Builds the project from |
|---|---|
| `create_project` | A compact task list you write: the best way to turn a description, an outline or a conversation into a plan. |
| `import_mspxml_project` | An MS Project XML document (hierarchy, durations, milestones, dependencies, notes), as File > Import XML… does. |
| `import_csv_project` | A CSV file's rows, as File > Import CSV does. |
| `import_xlsx_project` | One worksheet of an Excel workbook (`sheet_name`), as File > Import Excel Workbook does. |

When you are writing the plan yourself, use `create_project`, never
generated MS Project XML: the same plan is about ten times smaller, and
the tool checks it as it builds.

## Where the project goes

Every one of these tools takes `open_in_app`:

- `open_in_app: true` opens the project in FlexiGantt as a new untitled
  window, as File > Import does, and returns its `open_project_id`. Prefer
  this: the user sees the plan at once, nothing large passes through your
  context, and you can read it straight away with the `flexigantt` skill's
  tools (`list_tasks`, `get_critical_paths`, `get_status`). The window is
  unsaved until the user saves it.
- Otherwise the result carries the `.flg` file as an embedded binary
  resource (base64 in `resource.blob`). Decode it and save it where the
  user asks, with a `.flg` extension; the server never writes it to disk.

## `create_project`

```json
{
  "project_name": "Office move",
  "start_date": "2026-12-01",
  "notes": "Move the team to the new floor over the holidays.",
  "open_in_app": true,
  "tasks": [
    { "id": "prep", "name": "Preparation" },
    { "id": "survey", "name": "Survey new floor", "parent": "prep", "effort": 2, "resource": "Dana" },
    { "id": "order", "name": "Order furniture", "parent": "prep", "effort": 1, "predecessors": ["survey"] },
    { "id": "deliver", "name": "Furniture delivery", "parent": "prep", "effort": 10,
      "calendar_days": true, "predecessors": ["order"] },
    { "id": "pack", "name": "Pack desks", "effort": 3,
      "predecessors": [{ "id": "deliver", "type": "SS", "lag": 5 }] },
    { "id": "done", "name": "Team moved", "type": "milestone",
      "predecessors": ["pack", "deliver"], "deadline": "2027-01-08",
      "notes": "Everyone at a working desk on the new floor." }
  ]
}
```

- `tasks` are listed in display order. Each needs a unique `id` (your own
  key) and a `name`. `parent` names the summary it sits under; a task with
  children becomes a summary, and its own effort is ignored.
- `effort` is in person-days (default 1). A task's length is effort divided
  by its resource's velocity, which starts at 1, so effort 3 lasts 3
  working days.
- `predecessors` are task ids (Finish-to-Start), or
  `{id, type: FS|SS|FF|SF, lag}` with `lag` in working days.
- `calendar_days: true` makes a task run on every calendar day, weekends and
  holidays included. Use it for waits such as delivery, drying or approval
  turnaround, not for work. It does not apply to milestones or summaries.
- `start_no_earlier_than` (YYYY-MM-DD) holds a task back. `deadline` applies
  to milestones only and drives their buffer and health. A milestone takes
  one or the other, not both.
- `resource` is a person's name; new names become resources.
- `notes` on a task or on the project are Markdown: acceptance criteria,
  assumptions, links.
- `start_date` defaults to today, and nothing starts earlier.

FlexiGantt schedules the project itself (critical path, float, buffers).
Do not compute or pass dates for tasks, apart from `start_no_earlier_than`.

For a very large plan, write the same JSON object to a file and pass its
absolute path as `file_path` instead of the fields (see *Files* below).

The result reports `project_name`, `task_count`, and `notices` (things the
builder adjusted or ignored). Tell the user about any notices.

## Import a file

- **MS Project XML:** `import_mspxml_project`. `project_name` is optional
  and defaults to the XML's own. Task `<Notes>` are read as Markdown. The
  result reports `task_count` and `notes_count`.
- **CSV / Excel:** `import_csv_project` / `import_xlsx_project` need the
  column mapping: `name_column`, `start_column` and `finish_column` (header
  text, all required), plus an optional `notes_column`. Excel also takes
  `sheet_name`. `allow_non_working_days` (default false) matches the import
  dialog's checkbox: off moves a weekend or holiday row forward, on marks
  the task to allow non-working days. `project_name` defaults to
  "Imported Project". If you don't know the headers, read the file's first
  row before calling.
  The result reports `imported_count`, and per-row `issues` and `notices`.
  Report the issues to the user; never drop them silently.

## Files

Each import, and `create_project`'s `file_path`, takes the file one of
three ways:

- `file_path`: an absolute path on this Mac. Best for anything large. The
  first time, FlexiGantt asks the user to allow that folder; after that, the
  folder stays allowed (Settings > MCP Server > Folders).
- `file_text`: the contents as text (CSV, XML, JSON).
- `file_data`: the contents base64-encoded (needed for `.xlsx`).

If the user declines, the call fails with `FOLDER_NOT_AUTHORIZED`. Say
that FlexiGantt asked for access to the folder, and that the user can allow
it in Settings > MCP Server > Folders, or send the contents as `file_text`
or `file_data` instead. Do not retry in a loop, and do not copy the file
somewhere else to get around the prompt.

## Response envelope and errors

Every tool returns `schema_version`, `ok`, and `result` on success. On
failure it returns `ok: false` and an `error` with a stable `code` and a
`message`, for example `MAPPING_INCOMPLETE`, `MSPXML_PARSE_FAILED`,
`JSON_INVALID`, `FILE_DATA_MISSING`, `FILE_PATH_NOT_ABSOLUTE`,
`FOLDER_NOT_AUTHORIZED`. Pass the code and message on to the user, and fix
the input only when the message says what is wrong.

## Afterwards

Once the project is open (`open_in_app: true`), use the `flexigantt` skill
to check it: `get_critical_paths` for the driving sequence, `get_status` for
milestone health against the deadlines you set. To explain how FlexiGantt
schedules (effort, velocity, links, calendar days), read the User Guide
with `get_help`.
