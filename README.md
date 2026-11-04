# Avo MCP

mcp-name: io.github.avohq/avo

The **Avo MCP server** (`https://mcp.avo.app/mcp`) lets Claude, Codex, Cursor, Gemini, and other MCP clients read and edit your [Avo](https://www.avo.app) tracking plan. This repo also ships [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) and a Claude Code plugin that guide effective use of the server.

Each skill is a self-contained directory with a `SKILL.md` (instructions + metadata) and bundled resources. Claude loads them automatically when relevant to guide effective use of the Avo MCP server's tools (`describe_tool`, `search`, `get`, `save_items`, `workflow`, and the branch tools).

## Installing the Avo MCP server

The Avo MCP server is a remote, OAuth-protected HTTP server at `https://mcp.avo.app/mcp`. Add it to your MCP client:

**Claude Code (CLI)**

```bash
claude mcp add --transport http avo https://mcp.avo.app/mcp
```

You'll sign in through the browser on first use. The first time you make a change, you'll also get a one-time read → write re-auth prompt.

**Claude.ai / Claude Desktop**

Add a custom connector pointing at `https://mcp.avo.app/mcp` (Settings → Connectors → Add custom connector). Authenticate when prompted.

**Cursor and other MCP clients**

Add a remote (streamable HTTP) MCP server to your client's config:

```json
{
  "mcpServers": {
    "avo": { "type": "http", "url": "https://mcp.avo.app/mcp" }
  }
}
```

> If you install the Claude Code **plugin** (below), it registers this server for you — these steps are for connecting the server directly (other clients, or Claude Code without the plugin).

## Skills

### `data-designer` — Work with an existing Avo tracking plan

Explore, search, and design against an existing Avo tracking plan: look events / properties / metrics up by name, id, meaning, or structural filter; review, prioritize, and implement branches; design tracking for a new feature or PRD against what already exists; make small branch edits; and combine Avo with a connected data MCP (Amplitude, Mixpanel, PostHog, BigQuery, Snowflake, Databricks, Redshift) to diagnose tracking gaps.

- Path: [`skills/data-designer/`](skills/data-designer/)

### `data-designer-new-plan` — Bootstrap a brand-new Avo tracking plan

Bootstrap a tracking plan from scratch in an empty or near-empty workspace: run a purpose meeting (problems, goals, metrics, key funnels), agree a naming convention, and seed the first categories and `start → milestone → complete` events with event / user / system properties and constraints — then create a branch.

- Path: [`skills/data-designer-new-plan/`](skills/data-designer-new-plan/)

### Use-case skills

Each of these packages one job teams ask for again and again. They build on `data-designer` and produce a named output.

| Skill | What it does | Output |
|---|---|---|
| [`import-tracking-plan`](skills/import-tracking-plan/) | Brings an existing plan into Avo from a spreadsheet, a CSV export (Amplitude, Mixpanel, Segment, Avo), spec docs, Jira tickets, or another workspace | A branch + a reconciliation report |
| [`design-tracking-from-prd`](skills/design-tracking-from-prd/) | Turns a PRD, Figma file, or feature brief into tracking that fits your existing plan | A design proposal, then a branch |
| [`bulk-edit-tracking-plan`](skills/bulk-edit-tracking-plan/) | Applies one change across many items: name mappings, per-event object splits, PII, owners, custom fields, allowed values | A branch + a before/after summary |
| [`branch-implementation-handoff`](skills/branch-implementation-handoff/) | Hands a branch to developers: where each event fires, the full payload, and the code per platform | A handoff page + an optional Jira, Linear, or GitHub ticket |
| [`review-tracking-branch`](skills/review-tracking-branch/) | Reviews a branch against your rules before humans do, and lists what's waiting for your review | Findings per item + a reviewer summary |

## Use as a Claude Code plugin

This repo is also a Claude Code plugin marketplace. Installing the `avo` plugin bundles all the skills **and** registers the Avo MCP server (`https://mcp.avo.app/mcp`) in one step:

```bash
claude plugin marketplace add avohq/avo-mcp
claude plugin install avo@avo-mcp
```

The plugin's layout:

- `.claude-plugin/marketplace.json` — the marketplace manifest (lists the `avo` plugin)
- `.claude-plugin/plugin.json` — the `avo` plugin manifest
- `.mcp.json` — registers the Avo MCP server
- `skills/` — the skills, auto-discovered when the plugin is installed

## Install skills only (no plugin)

To use the skills without the plugin / MCP registration, copy a skill directory into your personal or project skills folder:

```bash
cp -R skills/data-designer ~/.claude/skills/data-designer
cp -R skills/data-designer-new-plan ~/.claude/skills/data-designer-new-plan
```

The use-case skills refer to `data-designer` for tool use and design principles, so copy it alongside any of them.

## Writing model

All writes happen on a branch, never on `main`; the MCP never merges — merging a branch into `main` is always a human step in the Avo web app.

The tool definitions the server sends on connect are kept short so that every MCP client sees every tool. The fields each item type accepts are fetched on demand: call `describe_tool()` for an overview of every tool and `describe_tool(tool:"save_items", type:"<type>", op:"<op>")` for the fields of an item type before writing it. Each `save_items` item has the shape `{op, type, id?, name?, tempId?, fields?}` with snake_case item types (`event_variant`, `property_bundle`, `group_type`, …); the camelCase spellings still work but are deprecated.

## Source

This repo is the source of truth for these skills. Avo's internal tooling syncs them from here (there's no build step to run in this repo). For the canonical, always-current Avo MCP tool list, see the [Avo MCP tools reference](https://www.avo.app/docs/reference/avo-mcp/tools).

## License

The contents of this repository — the Agent Skills in `skills/` and the plugin manifest and configuration — are released under the [MIT License](./LICENSE).

This covers **only the files in this repository**. It grants no rights to the Avo MCP server (`https://mcp.avo.app/mcp`), the Avo web application, Avo SDKs, or any other Avo software or service — these are separate works, governed by Avo's [Terms of Service](https://www.avo.app/l/terms) and your agreement with Avo.
