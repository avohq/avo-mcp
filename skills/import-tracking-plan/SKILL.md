---
name: import-tracking-plan
description: Import an existing tracking plan into Avo through the Avo MCP — from a spreadsheet, a CSV export (Amplitude, Mixpanel, Segment, Avo), an Avo JSON Schema, spec documents, Jira tickets, a wiki page, or another Avo workspace. Produces an Avo branch with the imported events, properties, and variants plus a reconciliation report of what came in, what was mapped onto existing items, and what needs a human decision. Triggers on "import our tracking plan", "get our spreadsheet into Avo", "migrate from Amplitude / Mixpanel / Segment", "load these specs into Avo", "copy events from workspace X to workspace Y", "we have our plan in a Google Sheet". For designing new tracking from a PRD use design-tracking-from-prd; for changing items that are already in Avo use bulk-edit-tracking-plan.
---

# Import a tracking plan into Avo

Most teams arrive with a plan that already exists somewhere: a spreadsheet, an analytics tool's export, a folder of spec docs, Jira tickets, or another Avo workspace. This skill gets that plan onto an Avo **branch** — never main — and tells the user exactly what happened to every row.

**What you produce:**

1. An Avo branch containing the imported items.
2. A reconciliation report: counts in vs. counts on the branch, items reused instead of duplicated, items renamed or name-mapped, and open decisions.

The MCP never merges. The user reviews the branch and merges it in the Avo web app.

If the `data-designer` skill is installed, its **Tracking design principles** and **Cross-cutting policies** apply here too. Field shapes change over time: call `describe_tool` before the first write of each item type instead of relying on examples in this file.

---

## Step 1: Start the session

1. `list_workspaces`, then set the target workspace (ask if more than one fits). Note the user's role: the native import (`workflow` `import`) requires **Admin**. Editors can still use the agent-driven path.
2. List every event on main: `search` with `itemType: "event"`, `branch: "master"`, and `maxResults: 500`, following `nextPageToken` (repeating every original parameter) through the last page. A branch scope on its own is a valid listing, and it includes events that aren't attached to any source. Keep this list for Step 4. No events means a straight import; otherwise you must reconcile against what is already there (Step 4).

## Step 2: Pick the import path

Look at what the user gave you and choose one path. Say which one and why.

| Input | Path |
|---|---|
| CSV export from Amplitude, Mixpanel, Segment, or Avo; a spreadsheet saved as CSV with one row per event/property | **Native import** — `workflow` with `action: "import"`, `format: "csv"`. The importer auto-detects these formats. |
| An Avo JSON Schema document | **Native import** — `format: "json_schema"`. |
| Spec docs, Jira tickets, wiki or Notion pages, Google Docs, screenshots, code, a spreadsheet with a layout the importer won't recognize | **Agent-driven import** — read the material, build the item list yourself, write it with `save_items`. |
| Another Avo workspace | **Agent-driven copy** — read the source workspace with `search`/`get`, write to the target with `save_items`. IDs are per workspace: never reuse an id from the source workspace; resolve everything by name in the target. |

Use the native path only for files that really are exports from those tools (or Avo's own CSV export). The importer's CSV layout isn't documented for hand-built files, so a custom spreadsheet goes agent-driven. If unsure, ask for one sample row and decide.

## Step 3: Read the rules

`get` with `type: "workspace_config"`. Note the naming format and casing, and which rules are **Enforce** (they block the branch at review) versus **Warning**. Typical blockers for an import: missing descriptions, `any`-typed properties, non-matching names.

## Step 4: Reconcile before writing

Build an item list from the source, then for each item decide: **create**, **reuse an existing item**, or **needs a decision**.

- **Recon with both lenses** for non-empty workspaces: a semantic `search` by meaning and a structural filter `search` (by `eventNames`, `properties`, `categories`). Never create a duplicate of something that exists. Exact-name lookups (`get` by name, the `eventNames` filter) are case-sensitive, so "checkout started" won't find "Checkout Started": compare names case-insensitively against the full listing from Step 1, and use semantic search for near matches. Semantic hits can come from other people's unmerged branches: only reuse items whose `Branch` is `master` (main), and confirm each with `get`.
- **Names that break the workspace convention.** Imported names are usually what the code sends today. Do not silently rename them. If the naming rule is **Enforce** (including an allowed-words list), keeping the old name will block the branch at review, so the usual answer is: rename to the convention and add a **name mapping** so destinations keep receiving the old name. If it is only a Warning, offer both. Workspaces with building-block naming (a format like `<OBJECT>_<ACTION>` in `workspace_config`) take the name's parts through `nameComponents` on create; check `describe_tool`.
- **Missing descriptions.** If descriptions are an Enforce rule, draft them in plain language (see data-designer's description principles) and mark them as drafted in the report so a human checks them.
- **Unknown types.** Infer from sample values or column hints. Don't use `any` when the workspace enforces against it; ask instead.
- **Allowed values.** Import them only when the source lists them. Never invent values.
- **Sources and destinations.** Ask which apps and platforms (sources) send these events and where they go. A plan exported from one tool (e.g. Amplitude) says nothing about the workspace's other destinations; don't assume.
- **Object properties.** Model nested fields as child properties (`nestedProperties` on create) rather than flattening them.

Show the user a preview before any write:

```
Import preview — <source name>
  Events:     42 to create · 6 reuse existing · 2 need a decision
  Properties: 118 to create · 21 reuse existing · 3 need a decision
  Variants:   7 to create
  Decisions:
    1. "signup_done" → keep, or rename to "SIGNUP_COMPLETED" + name mapping "signup_done"?
    2. "price" has values like "12.99" and "free" — string or float?
```

Wait for approval.

## Step 5: Write to a branch

1. `workflow` with `action: "create_branch"`. Put the source in the branch description ("Imported from <file/workspace> on <date>").
2. Warn once that the first write opens a browser prompt to grant write access.

**Native import:**

1. `describe_tool(tool: "workflow", action: "import")` for the current payload shape.
2. `workflow` with `action: "import"`, the `branchId`, `format`, and `payload`.
3. `importMethod`: use `add_only` (the default) unless the user explicitly wants existing items overwritten (`add_and_update`). **Never use `add_update_and_remove`** unless the user confirms the file is a *complete* export of the whole plan — it removes anything missing from the file.
4. Rows you decided to **reuse** must be removed from the file before a native import, or the importer will create or update them. Removing rows makes the file partial, so never combine it with `add_update_and_remove`. There is no dry run, so double-check the file against the preview first.
5. Apply the decisions from Step 4 that the CSV can't express (renames, name mappings, drafted descriptions) with `save_items` afterwards, on the same branch.

**Agent-driven import:**

1. `describe_tool(tool: "save_items", type: "<type>", op: "<op>")` for each type and op before its first write. `create` and `update` take different fields: on update, scalar edits go in `fields.set`. Each item has the shape `{op, type, id?, name?, tempId?, fields}`; type-specific fields such as `nestedProperties` or `nameMappings` go inside `fields`.
2. Order the batches so references resolve:
   - Properties first (child properties before or alongside their object parents), each with a `tempId`.
   - Events next, referencing properties as `$tmp:<tempId>` within the same batch, or by real id from an earlier batch's result.
   - Event variants after their base events exist.
   - Categories last: attaching a category to an item created **in the same batch** is rejected, so attach categories in a follow-up `update` batch.
3. At most 50 items per `save_items` call. Capture every returned id; later batches need them.
4. Skip `owner` and `stakeholders` during an import. They are **branch-independent**: they take effect workspace-wide immediately and are not rolled back if the branch is discarded. Offer to set them after the user has reviewed the branch.

## Step 6: Verify by reading back

Don't report success from the write responses alone.

1. Filter-mode `search` on the branch (`branch: <branchId>`) for every listable type the preview contains: `itemType: "event"`, `"property"`, and `"event_variant"` or `"metric"` if the import had them. Page with `pageToken` until done, repeating every parameter on each page, not just the token.
2. List sources and destinations (`search` with `itemType: "source"` / `"destination"`), and `get` each category and property bundle from the preview by name.
3. Compare everything with the preview, type by type. Investigate any gap before reporting.
4. `get` a few representative items (one object property, one variant, one event with many properties) on the branch and check them against the source, including their category and source attachments.

## Step 7: Report

```
Imported <source> into branch "<branch name>" — <link to branch in Avo>

  Events:     42 created · 6 reused existing
  Properties: 118 created · 21 reused existing
  Variants:   7 created
  Renamed with name mapping: 2 (listed below)
  Descriptions drafted by the agent: 37 — please review
  Not imported: 3 rows (listed below, with the reason)

Next: review the branch in Avo, then merge it there. The MCP never merges.
```

---

## Gotchas

- **Agents skip reconciliation in a non-empty workspace** and create duplicates. Recon is required for every item, not a sample.
- **Agents "fix" imported names to the convention silently.** That breaks every dashboard built on the old name. Rename only with the user's approval and a name mapping.
- **`add_update_and_remove` on a partial file** deletes properties from events. Treat it as destructive.
- **A CSV import only sets name mappings on events it creates.** On events that already exist it skips them. Use `save_items` updates for those (see bulk-edit-tracking-plan).
- **Category attach in the same batch as the create** fails. Attach in a follow-up batch.
- **`workflow` import doesn't check review status** and takes no `onReviewedBranch`: it writes to any open branch, including one someone is reviewing. Always import onto the branch you created in Step 5.
- **Very large plans.** Work in batches of 50 and report progress between batches; don't try to hold the whole plan in one call.
- **Copying between workspaces.** Sources, destinations, and categories may not exist in the target. Create or map them first, then events.
