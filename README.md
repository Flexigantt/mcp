# FlexiGantt MCP & Agent Skills

[FlexiGantt](https://flexigantt.com) is a native Mac app for classical,
plan-driven project management: a work breakdown structure, dependencies,
critical path (CPM) scheduling with float, milestones and deadline buffers,
baselines, and Gantt charts and roadmaps built from the same plan. This
repo connects an AI agent (Claude Code, Codex) to that plan through
FlexiGantt's local MCP server, and bundles the Agent Skills that teach the
agent to use it: building a new plan or importing one, answering questions
about a live schedule, planning the week, exporting Gantt and Roadmap views,
and writing status reports.

MCP-only, no CLI required. The MCP tools can create a new project, but
never change an existing one. FlexiGantt's experimental CLI skills
(including scenario/what-if modeling, which needs that kind of write
access) ship separately with a direct FlexiGantt download, not through this
marketplace.

FlexiGantt's MCP server is off by default (Settings > MCP) and only ever
listens on `127.0.0.1`. Agents sign in with OAuth: the first time one
connects, your browser opens and FlexiGantt asks you to allow it. There's no
token to copy, and you can revoke an agent any time in Settings > MCP Server >
Connection.

## What's new in plugin 1.3.0

**Build a plan from a conversation.** The new `flexigantt-create` skill
teaches the agent `create_project`: describe the work and the agent writes
a compact task list (hierarchy, effort, dependencies with lag, milestones,
deadlines, calendar-day waits, resources, Markdown notes). FlexiGantt
schedules it and opens it as a new window.

**Import a file.** The same skill covers `import_csv_project`,
`import_xlsx_project` and the new `import_mspxml_project` for MS Project XML.
An agent can pass a file by path; the first time, FlexiGantt asks you to
allow its folder (Settings > MCP Server > Folders).

**Plan the week.** The `flexigantt-weekly-plan` skill now ships here too:
"what's the plan for this week?" from the Iterations view, and a ranked
list of the milestones that need attention.

**Answers from the User Guide.** `get_help` lets the agent read FlexiGantt's
User Guide, so it explains how the app schedules instead of guessing.

**Status reports use the current fields.** `flexigantt-status-report`
reads Schedule status, Health and Stale separately, with buffer units,
matching what `get_status` returns. It replaces the single red/amber/green
signal of earlier versions.

## What's new (FlexiGantt 1.15, plugin 1.2.0)

**Start a plan from a template.** File > New Project now opens a template
browser: Agile Development, CRM Implementation, a cloud migration and Home
Renovation, each starting today with its tasks, dependencies and milestones
in place. An agent can then read and explain the new plan right away.

**Sign in instead of copying a token.** Agents connect with OAuth: the first
time, your browser opens and FlexiGantt asks you to allow the agent. After
that it stays signed in and renews its access on its own. Each time
FlexiGantt starts, it asks you once more before a signed-in agent connects.
Settings > MCP Server > Connection lists every allowed agent, with Revoke.

**Exports are the app's own views.** `export_gantt_pdf` draws the Gantt
chart as File > Export > PDF does: task table, timeline, bars and dependency
lines. `export_roadmap_pdf` and `export_roadmap_pptx` produce the Roadmap's
real slides, every slide, and the PowerPoint deck is editable native shapes.

**Agents can set the view.** Exports follow the open project's window
settings, and any of them can be overridden per call without touching the
window:
- Roadmap: slide `layout` (16:9 or 4:3), one `slide` or all of them, the
  exact `tasks` to show, `color_scheme`, `time_scale`, and each inspector
  switch (float, deadline buffer, progress, status, task column, align to
  today, row bands). The result reports `slide_count`.
- Gantt: `time_scale`, `show_dependencies`, `show_float`.

**Also in the app:** with no slide selected, the Roadmap's PDF and
PowerPoint exports hold every slide, and float outlines in PowerPoint
exports now stay on their bars.

**Upgrading from 1.1.0:** this version needs FlexiGantt 1.15 or later.
Remove the old `export FLEXIGANTT_MCP_TOKEN=…` line from your shell profile,
then sign in as below. Codex users who added the server with
`--bearer-token-env-var` should remove it (`codex mcp remove flexigantt`)
and add it again with the command FlexiGantt shows in Settings > MCP Server >
Agents.

## Install

**Claude Code**

```bash
claude plugin marketplace add Flexigantt/mcp
claude plugin install flexigantt
```

**Codex**

```bash
codex plugin marketplace add Flexigantt/mcp
codex plugin install flexigantt
```

Then sign in once: in Claude Code, run `/mcp`, pick **flexigantt** and choose
**Authenticate** (Codex: `codex mcp login flexigantt`). Your browser opens,
FlexiGantt asks you to allow the agent, and it stays signed in from then on.

Enable the MCP server in FlexiGantt itself first (Settings > MCP > Enable MCP
Server) — the agent connects to whichever project you have open, live.

## What's in here

- `skills/flexigantt` — the core skill: read a live project, explain the
  app from its User Guide, and export Gantt/Roadmap views via MCP tools.
  Read-only.
- `skills/flexigantt-create` — build a new project from a task list, or
  import one from CSV, Excel or MS Project XML, and open it in the app.
- `skills/flexigantt-weekly-plan` — this week's plan from the Iterations
  view, and the milestones to focus on.
- `skills/flexigantt-status-report` — narrative status-report authoring,
  MCP-only.
- `.mcp.json` — the MCP server declaration installed alongside the skills.
