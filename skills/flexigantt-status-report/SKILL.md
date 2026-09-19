---
name: flexigantt-status-report
description: Create current-state executive, milestone-health, and HTML status reports from a FlexiGantt project open in the app, with live reads and exports via the bundled MCP tools. Use for reporting the live plan — "give me the updated roadmap", "how do I explain this upward" — not for hypothetical scenario comparisons or schedule mutations.
---

# FlexiGantt Status Reports (MCP)

FlexiGantt Status Report MCP Skill Version: 1
Requires a FlexiGantt build with Settings > MCP enabled and this agent
connected to it.

Use this skill when the deliverable is a narrative project status report
rather than a raw schedule inspection or one-off export. Report from the
live project and preserve FlexiGantt's own milestone and dependency-gate
signals.

For hypothetical changes or comparisons between schedule copies, use the
`flexigantt-what-if` skill instead — it requires the experimental
`flexigantt` CLI skill and is not available through MCP alone.

## Read authoritative report inputs

Re-read the selected project immediately before generating the report —
never reuse schedule facts from an earlier turn after the user says the
plan changed.

Call `list_open_projects` to get the project's `open_project_id`, then:

```
get_project        (open_project_id)
list_tasks         (open_project_id)
get_status         (open_project_id)
get_critical_paths (open_project_id)
```

This reads the live in-memory document, including unsaved edits — exactly
what "re-read before reporting" requires, with no file path to guess and
nothing to save first. If `list_tasks` or `get_status` will be read more
than once in this session, fetch the `flexigantt://<open_project_id>/tasks`
and `.../status` MCP resources instead of repeating the tool call.

If the project is not open in the app, or these tools are not available in
this session, point the user to FlexiGantt > Settings > MCP — there is no
CLI fallback available in this skill.

Use the results as follows:

- `get_project`: project identity, schedule start, task count, and source
  path.
- `list_tasks`: WBS hierarchy, dates, completion, task status, float, and
  predecessors.
- `get_status`: milestone and dependency-gate health, current forecast,
  baseline, signal, variance, progress, buffer, and impact.
- `get_critical_paths`: the current driving sequences and their forecast
  dates.

If the report includes a data-as-of date and the schedule does not provide
one, use the actual report-generation date and label it as such. Do not
present it as a schedule status date.

## Export the schedule views

`export_roadmap_pdf`, `export_gantt_pdf`, and `export_roadmap_pptx` are
available as MCP tools — call them with the same `open_project_id` used for
the reads above, and save the returned `resource.blob` bytes to the paths
this report needs. This reads the live, possibly-unsaved document, same as
the tools above.

Export a Roadmap PDF for a status report. Add a Gantt PDF when the audience
benefits from detailed task timing. Use `export_roadmap_pptx` when the
audience needs an editable slide rather than a fixed PDF — a PM finishing a
status report almost always wants to drop it into their own deck template,
not a locked image.

Require success, report the absolute output paths, and verify that each
output exists and is nonempty. When practical, render or open the exports
and check that labels, task bars, and milestone markers are legible and
unclipped.

For an HTML deliverable, place the HTML and its linked PDFs where relative
links remain valid. Link the Roadmap prominently and either embed it with
an `<object>` element or include a preview image with a PDF fallback link.
Keep the detailed Gantt available as a secondary link.

## Interpret controls without inventing meaning

Present FlexiGantt status fields faithfully:

- Preserve each control's `name`, `control_type`, and `signal` exactly.
  Styling may map red, amber, and green to accessible colors, but must not
  rename or soften the signal.
- Show `planned_finish` as the current forecast and `baseline_finish` as
  the baseline when present.
- Show `variance_days` with its sign and describe the unit as days
  reported by FlexiGantt unless the project's calendar semantics are
  known.
- Show `critical`, `remaining_buffer_days`, and `critical_buffer_days`
  when they help explain the signal.
- Show control `progress` as a percentage and respect its `progress_mode`
  label.
- Resolve `impacted_control_ids` and `impacted_task_ids` to names from the
  same live `list_tasks` output before summarizing them. Do not expose raw
  UUIDs to a report audience unless requested.
- Include dependency gates as well as milestones when either array is
  nonempty. Do not silently drop healthy controls.
- If a baseline, variance, or buffer value is null, display "Not set" or
  omit the metric; do not convert null to zero.

Separate reported facts from interpretation. It is acceptable to state
that a red, critical, zero-buffer milestone requires attention. Label
proposed recovery actions or management decisions as recommendations
rather than schedule facts.

## Calculate phase and overall progress carefully

Summary-task `percent_complete` may not aggregate its children. Do not use
a summary row's value as phase progress unless the output confirms that it
is aggregated.

When phase progress is useful, derive it from non-summary descendants and
state the method. A reasonable default when effort exists is:

```text
phase progress = sum(task effort * task percent complete) / sum(task effort)
```

Exclude summary rows and zero-effort milestones from the denominator. If no
usable effort exists, use a clearly labeled task-count method or omit the
calculation. Apply the same method consistently to overall delivery
progress. Do not present a calculated value as a native FlexiGantt metric.

## Recommended HTML content

Build a standalone, responsive, print-friendly report with the smallest
structure that communicates the schedule clearly:

1. Project name, report-generation date, and schedule window.
2. Overall signal and headline metrics such as progress, current
   completion forecast, and baseline variance.
3. A short executive summary grounded in the current read.
4. Phase progress with the calculation method disclosed.
5. A complete milestone and dependency-gate status section.
6. The exported Roadmap view, with a direct PDF link and a fallback when
   embedding is unavailable.
7. A link to the detailed Gantt export when generated.
8. Current critical-path focus and clearly labeled recommendations or
   decisions needed.
9. A note that schedule values came live from FlexiGantt via MCP.

Prefer semantic HTML, accessible color contrast, visible labels in
addition to color, meaningful link text, mobile behavior, and print CSS.
Avoid external fonts or hosted assets when the report must remain
portable.

## Validate before delivery

Before claiming completion:

- reconfirm that all reads and exports returned `ok: true`;
- check that the HTML references the intended Roadmap and Gantt files and
  every relative target exists;
- check milestone and dependency-gate values against the latest
  `get_status` response;
- check phase and overall calculations independently from task data;
- render the HTML when tooling is available and inspect a desktop and
  narrow viewport;
- verify that embedded-PDF failure leaves a usable direct link;
- report the final HTML and exported PDF paths.

Do not claim the HTML itself was generated by FlexiGantt. Identify it as a
report based on FlexiGantt data and exports.
