---
name: bulk-edit-tracking-plan
description: Make the same change across many existing items in an Avo tracking plan through the Avo MCP — add name mappings or aliases in bulk, split one shared object property into per-event versions that send the same name, declare PII on every property, reassign owners and stakeholders after a reorg, set custom fields or tags, add or strip a property across many events, move properties from a source to a gateway, add allowed values. Produces an Avo branch with the change set and a before/after summary. Triggers on "for all events", "every property that", "in bulk", "add aliases to", "name map these", "mark all PII", "the team was reorganised", "different required fields per event for the same object". For a one-off edit to a single item, the data-designer skill is enough; to bring in a plan from outside Avo use import-tracking-plan.
---

# Bulk-edit a tracking plan

Teams often need the same change applied to dozens or hundreds of items: a destination needs different names, a privacy review needs PII flags everywhere, a team reorg moves ownership, one shared object has to behave differently per event. In the Avo web app that is one item at a time. Through the MCP it is one change plan and a few batches.

**What you produce:**

1. An Avo branch with the change set (except branch-independent fields — see below).
2. A before/after summary: how many items matched, what changed on each, and anything skipped.

The MCP never merges. The user reviews and merges in the Avo web app.

If the `data-designer` skill is installed, its policies and design principles apply. Call `describe_tool(tool: "save_items", type: "<type>", op: "<op>")` before the first write of each type and op (`update`, or `create` for the recipes that add new properties): the two ops take different fields. field shapes in this file are illustrations, `describe_tool` is the contract.

---

## Step 1: Select the items exactly

1. `list_workspaces` and set the workspace.
2. Translate the user's selection into a **filter-mode** `search` (no `query`): `tags`, `categories`, `sources`, `destinations`, `eventNames`, `properties`, `nameMapping`, `pii`, `customField`, `owners`, `stakeholders`. Values in one array are OR'd; different keys are AND'd.
3. Page through **every** result: pass `pageToken` and repeat every other parameter (`workspaceId`, `itemType`, filters, `branch`, `maxResults`) on each page. A selection that stops at page one silently changes 10 items out of 300.
4. If the selection can't be expressed as filters ("all checkout-ish events"), use semantic search to propose a list, then have the user confirm the exact list. Semantic hits can come from other people's unmerged branches; keep only items whose `Branch` is `master` (main).
5. **Properties "in" a category can mean two things.** `itemType: "property"` with `categories` returns the properties assigned to that category directly. If the user means the properties sent on the category's events, that takes two steps: list the events (`itemType: "event"`, `categories`), then the properties on them (`itemType: "property"`, `eventNames`). Ask which one they mean when it isn't clear. That listing doesn't include system properties or the child fields of object properties; add those deliberately if the change should cover them.

State the selection back with a count: "47 properties with no PII declaration, across 12 categories."

**Check the premise before planning.** If items are already in the target state (every property is already snake_case), say so and stop; don't send no-op writes. If the selection contradicts what the user described ("every event that uses `products`" turns out to be one event), tell them what you found and ask before continuing.

## Step 2: Write a change plan and preview it

For each matched item, the exact change. Show a compact table and the total, for example:

```
Change plan — PII declaration (47 properties)
  email, phone_number, ...      → declared: Contact Information (12)
  user_id, device_id, ...       → declared: Personal Identifier (9)
  page_name, order_value, ...   → not PII (26)
  Needs a decision: 3 (listed)
```

Flag **branch-independent** changes before asking for approval:

> `owner` and `stakeholders` take effect **workspace-wide immediately** and are **not rolled back** if the branch is discarded. They also reference stakeholders that must already exist in Avo; the MCP can't create them.

Wait for approval. For owner/stakeholder changes, get explicit approval for that part specifically.

## Step 3: Apply on a branch

1. `workflow` with `action: "create_branch"`, with a description of the change ("Declare PII on all properties — privacy review Q4").
2. `save_items` with `op: "update"` items, **at most 50 items per call**. Each item has the shape `{op, type, id, fields}`; the fields named in the recipes below go inside `fields`. An `event_variant` has no `id`: its ids go in `fields` as `baseEventId` and `variantId`. Report progress between batches on large sets.
3. If a batch fails, stop and report which items failed and why. Don't retry the whole set blindly; some items in a failed batch may need a different shape.

## Step 4: Verify

1. Keep the ids of every item you selected in Step 1. A filter on the new state (e.g. `pii: {kind: "noPii"}`) can also return items that were already in that state, so it can't prove the batch worked on its own.
2. Re-run the Step 1 selection on the branch (`branch: <branchId>`) and check that every selected id now has the new value; `get` the ones the listing doesn't show the changed field for. Report any id that didn't change.
3. Report the before/after summary and the branch link.

---

## Recipes

### Gateway workspaces: select by what actually arrives, change it where it happens

If `search` with `itemType: "gateway"` returns any gateways, a destination can receive something different from what the plan says:

- **Selecting "everything that reaches X":** the `destinations` filter means *configured to send to X*, not *arrives at X*. Use `search` with `checkpoint: {kind: "destination", destinationId}` for what actually arrives (the two lists can differ a lot).
- **Before adding a mapping, check the gateway.** `get` the item you're mapping with the same checkpoint: the event for an event mapping, the property for a property mapping (an event arriving under the target name says nothing about its properties). If it already arrives under the target name ("sent as …"), the gateway output renames it and there's nothing to do.
- **Change it at the output, not on the event.** Renames and allow-lists for one destination usually belong on the gateway's output. Use `save_items` with `type: "gateway"`, `op: "update"`; call `describe_tool(tool: "save_items", type: "gateway", op: "update")` for the current fields. Event-level name mappings on top of a gateway rename create two names for one thing.

### Bulk name mappings (aliases)

Use `nameMappings` on `update` for events or properties: `{destination: {kind: "allDestinations"}, name: "<sent name>"}`, or `{destination: {kind: "destination", destinationId}, name}` for one destination. `name: null` removes a mapping. Resolve destination ids with `search` (`itemType: "destination"`).

A CSV import only sets name mappings on events it creates and skips events that already exist, so this recipe is the way to add mappings to an existing plan.

### One object per event, all sending the same name

Use this when a shared object property (e.g. `products`, `items`, `ecommerce`) needs different required/optional fields on different events.

**Where "optional" lives:** whether a child field is optional inside an object is stored on the **child property itself**, not on the parent's slot. Two object copies that share the same child property can't disagree, because the last write wins for both. So a child field that is optional on one event and required on another needs **its own copy too**.

1. `get` the existing object property and the events that use it. Note which child fields each event needs, and which are optional on that event. If only one event uses the object, a split isn't needed; just set the child's optional flag.
2. For each child field whose optional/required setting differs between events, create a copy of that child property with a distinct Avo name that follows the workspace's naming rules (e.g. `product_context_optional`), with `nameMappings` `{destination: {kind: "allDestinations"}, name: "product_context"}` so it's still sent under the original name.
3. Create one object property per event shape, with a distinct Avo name that follows the naming rules (e.g. `products_checkout`), and:
   - `nestedProperties`: the child slots, each `{propertyId, optionalWhenInObject?}`, pointing at the original children or the copies from step 2 (`$tmp:` refs work in the same batch).
   - `nameMappings`: `{destination: {kind: "allDestinations"}, name: "products"}`, so every destination still receives `products`.
   - A description saying which event it belongs to and why it differs.
4. If the original is a list of objects, set `isList: true` on each copy in a **follow-up** `update` batch; `isList` can't be set on create.
5. On each event: `removeProperties` the shared object, `addProperties` its per-event version.
6. Keep the shared property until the user decides to archive it, since other events may still use it.

Reading back can't confirm the optional/required setting of each child field yet. Check one in the Avo web app, or check the generated code snippet for a codegen source (`get` with `type: "branch"`, `include: ["code_snippets"]`, `sourceId`).

### PII

`pii` on a property `update`: `{kind: "declared", typeId: "<PII type id>"}`, `{kind: "notPii"}`, or `{kind: "unset"}`. `typeId` is optional on `declared` and must be one of the PII type **ids** listed in `get(type: "workspace_config")` (the MCP can't create PII types). Don't pass `typeId` with `notPii` or `unset`.

### Owners and stakeholders after a reorg

`owner: {action: "set", stakeholder: {stakeholderName}}` and `stakeholders: {add: [...], remove: [...]}`. Branch-independent, as above: confirm before writing, and tell the user the change is already live when you report.

### Custom fields, tags, and categories

`customFieldValues`, `addTags` / `removeTags`, `addCategories` / `removeCategories` (category names). Custom-field ids are in `get(type: "workspace_config")`.

### Add or remove a property on many events

`update` each event with `addProperties` / `removeProperties`. To change how the property behaves on specific events (presence, a pinned value, allowed values per event), use `eventConfigs` on the property `update` instead.

### Allowed values

`addAllowedValues` / `removeAllowedValues` on the property. To restrict values on one event only, use `eventConfigs` with `restrictAllowedValues`.

---

## Gotchas

- **`op: "update"` on a property changes it everywhere it is used.** If the user means "on this event only", use `eventConfigs` or an event variant, not a property edit.
- **Stopping at the first page of results.** Always page to the end and say the total.
- **Owner and stakeholder edits are live immediately**, branch or not. Never bundle them silently into a branch change.
- **Categories can't be attached to items created in the same batch.** Attach in a follow-up batch.
- **Renames break dashboards.** A bulk rename without a name mapping changes what destinations receive. Ask before renaming; add mappings when the sent name must stay.
- **Some writes depend on the workspace's plan:** name mappings, event variant creation, triggers, pinned properties, and gateways among them. A `paywall_blocked` error means the plan doesn't include that feature. Tell the user; don't retry.
- **Writing to a branch that's in review is rejected** unless the call passes `onReviewedBranch: {kind: "revertToDraft"}`, which sends the branch back to Draft and drops it out of review. Ask the user before passing it; usually a fresh branch is better.
- **Recurring jobs** (a nightly sync of tiers or owners) need a non-interactive login, which the MCP doesn't support yet. For a scheduled job, point the user to the Avo import API instead.
