# Avo MCP

The **Avo MCP server** (`https://mcp.avo.app/mcp`) lets Claude and other MCP clients read and edit your [Avo](https://www.avo.app) tracking plan. This repo also ships [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) and a Claude Code plugin that guide effective use of the server.

Each skill is a self-contained directory with a `SKILL.md` (instructions + metadata) and bundled resources. Claude loads them automatically when relevant to guide effective use of the Avo MCP server's tools (`search`, `get`, `save_items`, `workflow`, and the branch tools).

## Installing the Avo MCP server

The Avo MCP server is a remote, OAuth-protected HTTP server at `https://mcp.avo.app/mcp`. Add it to your MCP client:

**Claude Code (CLI)**

```bash
claude mcp add --transport http avo https://mcp.avo.app/mcp
```

You'll sign in through the browser on first use. The first time you make a change, you'll also get a one-time read → write re-auth prompt.

**Claude.ai / Claude Desktop**

Add a custom connector pointing at `https://mcp.avo.app/mcp` (Settings → Connectors → Add custom connector), or find **Avo** in the MCP Directory. Authenticate when prompted.

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

## Use as a Claude Code plugin

This repo is also a Claude Code plugin marketplace. Installing the `avo` plugin bundles both skills **and** registers the Avo MCP server (`https://mcp.avo.app/mcp`) in one step:

```bash
claude plugin marketplace add avohq/avo-mcp
claude plugin install avo@avo-mcp
```

The plugin's layout:

- `.claude-plugin/marketplace.json` — the marketplace manifest (lists the `avo` plugin)
- `.claude-plugin/plugin.json` — the `avo` plugin manifest
- `.mcp.json` — registers the Avo MCP server
- `skills/` — the two skills, auto-discovered when the plugin is installed

## Install skills only (no plugin)

To use the skills without the plugin / MCP registration, copy a skill directory into your personal or project skills folder:

```bash
cp -R skills/data-designer ~/.claude/skills/data-designer
cp -R skills/data-designer-new-plan ~/.claude/skills/data-designer-new-plan
```

## Writing model

All writes happen on a branch, never on `main`; the MCP never merges — merging a branch into `main` is always a human step in the Avo web app.

## Source

This repo is the source of truth for these skills, and they're also submitted to the Claude MCP Directory. Avo's internal tooling syncs them from here (there's no build step to run in this repo). For the canonical, always-current Avo MCP tool list, see the [Avo MCP tools reference](https://www.avo.app/docs/reference/avo-mcp/tools).
