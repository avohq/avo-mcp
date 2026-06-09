# Avo MCP Skills

[Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) for working with the [Avo](https://www.avo.app) tracking plan through the **Avo MCP server** (`https://mcp.avo.app/mcp`).

Each skill is a self-contained directory with a `SKILL.md` (instructions + metadata) and bundled resources. Claude loads them automatically when relevant to guide effective use of the Avo MCP server's tools (`search`, `get`, `save_items`, `workflow`, and the branch tools).

## Skills

### `avo-mcp` — Work with an existing Avo tracking plan

Explore, search, and design against an existing Avo tracking plan: look events / properties / metrics up by name, id, meaning, or structural filter; review, prioritize, and implement branches; design tracking for a new feature or PRD against what already exists; make small branch edits; and combine Avo with a connected data MCP (Amplitude, Mixpanel, PostHog, BigQuery, Snowflake, Databricks, Redshift) to diagnose tracking gaps.

- Path: [`avo-mcp/`](avo-mcp/)

### `avo-mcp-new-tracking-plan` — Bootstrap a brand-new Avo tracking plan

Bootstrap a tracking plan from scratch in an empty or near-empty workspace: run a purpose meeting (problems, goals, metrics, key funnels), agree a naming convention, and seed the first categories and `start → milestone → complete` events with event / user / system properties and constraints — then create a branch.

- Path: [`avo-mcp-new-tracking-plan/`](avo-mcp-new-tracking-plan/)

## Writing model

All writes happen on a branch, never on `main`; the MCP never merges — merging a branch into `main` is always a human step in the Avo web app.

## Source

These skills are maintained in Avo's internal monorepo (under `.rulesync/skills/`) and published here for the Claude MCP Directory. For the canonical, always-current Avo MCP tool list, see the [Avo MCP tools reference](https://www.avo.app/docs/reference/avo-mcp/tools).
