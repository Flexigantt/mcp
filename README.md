# FlexiGantt MCP & Agent Skills

Connects an AI coding agent to [FlexiGantt](https://flexigantt.com) — a native
Mac Gantt/Roadmap scheduling app — via its local MCP server, and bundles the
Agent Skills that teach the agent how to use it (reading a live project and
authoring status reports). MCP-only — read-only, no CLI required. FlexiGantt's
experimental CLI skills (including scenario/what-if modeling, which needs
write access these MCP tools don't have) ship separately with a direct
FlexiGantt download, not through this marketplace.

FlexiGantt's MCP server is off by default (Settings > MCP) and only ever
listens on `127.0.0.1`, authenticated with a bearer token generated on your
Mac. This repo never contains that token — you export it into your own shell
once, from the value FlexiGantt shows you in Settings > MCP.

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

Then, once (per shell profile — `~/.zshrc` or equivalent), so the MCP
connection can authenticate:

```bash
export FLEXIGANTT_MCP_TOKEN='<paste the token from FlexiGantt > Settings > MCP>'
```

Enable the MCP server in FlexiGantt itself first (Settings > MCP > Enable MCP
Server) — the agent connects to whichever project you have open, live.

## What's in here

- `skills/flexigantt` — the core skill: read a live project and export
  Gantt/Roadmap views via MCP tools. Read-only.
- `skills/flexigantt-status-report` — narrative status-report authoring,
  MCP-only.
- `.mcp.json` — the MCP server declaration installed alongside the skills.
