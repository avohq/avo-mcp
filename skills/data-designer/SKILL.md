---
name: data-designer
description: Work with an existing Avo tracking plan through the Avo MCP. Use to explore and search events, properties, and metrics; look items up by exact name or id; list items by structural filters; review and prioritize branches; design tracking for a new feature or PRD against the existing plan; make small branch edits (rename, allowed values, attach properties); implement a branch's changes in source code; and combine Avo with a data MCP (Amplitude, Mixpanel, PostHog, BigQuery, Snowflake, Databricks, Redshift) to diagnose tracking gaps. Triggers on "what events do we have for", "how is X tracked", "add tracking for", "design tracking for", "review the X branch", "what's on branch X", "implement the X branch for [source]", "look up [event/property/metric]", and analysis prompts like "where are users dropping off" or "how is [funnel] performing". If the workspace is empty or brand-new, use the data-designer-new-plan skill instead.
---

# Avo MCP

This skill teaches the agent how to use the Avo MCP server to read and write a customer's *existing* tracking plan. It is the playbook for exploring, searching, designing-against, reviewing, and implementing a plan that already has a meaningful taxonomy, plus the policies and gotchas that keep agents from breaking main.

If the workspace is empty or brand-new and the team is deciding what to track from scratch, use the **data-designer-new-plan** skill instead. Quick empty check: run `search` with `itemType: "event"` and no `query`; very few results means greenfield — switch to that skill.

The Avo MCP server runs at `https://mcp.avo.app/mcp` and exposes these tools:

Read tools: `health_check`, `list_workspaces`, `get`, `search`, `list_branches`, `get_branch_details`, `get_branch_implementation_guide`, `get_branch_code_snippets`.

`get` looks up a single item by id or exact name. It supports `type` values `event`, `property`, `metric`, `category`, `propertyBundle`, `source`, `destination`, `groupType`, `eventVariant`, and `workspaceConfig`. Use `type: "workspaceConfig"` to retrieve the workspace's tracking plan audit rules — i.e. the workspace-wide event naming, property naming, casing, and validation rules — before proposing any new events or properties.

Write tools (require the `write` scope): `workflow`, `save_items`.

The MCP never merges. Merging a branch to main always remains a human step in the Avo web app.

---

## Flow 1: Start session (bootstrap)

Runs once per session, on the first prompt of any kind. Required before any other workspace-scoped tool.

Trigger: any prompt.

Steps:

1. Call `list_workspaces`.
2. If the user named a workspace (by name or id) in their prompt: set that as the active workspace, respond "Workspace set to [Workspace Name]", and continue answering the original question.
3. Else if the user has access to exactly one workspace: set it active silently and continue.
4. Else: return a numbered list of workspaces the user has access to and ask which one they would like to use. Wait for the user to select, then set it active and continue.

Hold the active `workspaceId` in conversation state for the rest of the session. Every workspace-scoped tool needs it.

---

## Flow 2: Explore the tracking plan

Open-ended discovery by meaning.

Trigger prompts:
- "What events do we have for signup?"
- "How do we track a successful checkout?"
- "Is there anything that captures onboarding completion?"

Steps:

If Smart Search is enabled in the workspace:

1. `search` with `query` set to the user's phrasing (semantic mode). Default `itemType` is `event`; widen to other types when the user's phrasing implies it.
2. When the user's phrasing also constrains structurally (e.g. "events on the iOS source", "metrics in category Y", "events with the `product_id` property"), combine the semantic `query` with filter fields (`sources`, `categories`, `properties`, etc.) on the same `search` call to narrow the result set before paging through hits.
3. `get` on each promising hit by id.
4. Report back with the real names returned by `get`, not the semantic paraphrase. Ask if that answered the user's question.

If Smart Search is not enabled:

Return: "You don't have Smart Search enabled in your workspace, so I can only search by exact item names or IDs."
- If the user is a workspace admin: include a short guide on how to turn Smart Search on.
- If the user is not an admin: recommend they ask a workspace admin to enable it.

---

## Flow 3: Find by exact name or ID

Targeted lookup of one item.

Trigger prompts:
- "How is the Signup Completed event defined?"
- "What does the `product_id` property look like?"
- User pastes a name or id.

Steps:

1. `get` with the exact name or id.
2. If `get` finds a match: return the spec of the item in a format that answers the user's question. Ask if that answered their question.
3. If the user pasted multiple names or ids at once: prefer a single filter-mode `search` call (e.g. `itemType: "event", eventNames: ["A", "B", "C"]`) over calling `get` once per name. One call returns the whole set and respects `maxResults` / pagination.
4. If `get` does not find a match:
   - If Smart Search is on: `search` as a fallback (the miss is likely a slight mismatch).
     - If `search` returns at least one hit: `get` on the top hit's id, then return the spec.
     - If `search` returns no hits: tell the user no close match was found, and ask for an exact name/id or a more specific phrasing.
   - If Smart Search is off: tell the user the exact name or id wasn't found, and explain Smart Search would enable fuzzy fallback.

---

## Flow 4: Get a filtered list of items

Structured listing across the plan. Uses `search` in filter mode (no `query`, structural filters instead).

Trigger prompts and how each maps to filter fields:

- "What events is the `product_id` property on?" → `itemType: "event"`, `properties: ["product_id"]`
- "Which events have the `signup` category?" → `itemType: "event"`, `categories: ["signup"]`
- "Which events are name-mapped to `page_view`?" → `itemType: "event"`, `nameMapping: { kind: "matchesAny", names: ["page_view"] }`
- "Which properties are on the `Account Created` event?" → `itemType: "property"`, `eventNames: ["Account Created"]`
- "Which events are tracked from the iOS source?" → `itemType: "event"`, `sources: ["iOS"]`
- "Which events have variants?" → `itemType: "event"`, `includeVariants: true`

Steps:

1. Pick `itemType` from the user's phrasing: `event` (default), `property`, `metric`, `eventVariant`, `category`, or `propertyBundle`.
2. Translate the phrasing into one or more filter fields: `tags`, `categories`, `sources`, `eventNames`, `variantNames`, `properties`, `nameMapping`. Multiple values in one array are OR'd. Multiple keys are AND'd.
3. Call `search` with those filters and no `query` field. This is filter listing mode (cap 500, default 10). Set `includeVariants: true` when the user asks for events with their variants.
4. Return: a one-line summary of what was filtered for ("Events with the `product_id` property"), then the list of results.
5. If the result set hits `maxResults` and there is a `nextPageToken`, offer to paginate.

Notes:

- For metrics, `categories` and `sources` filters do not apply. Use `tags` or direct item-name lookup instead.
- `nameMapping` is a tagged object discriminated by `kind`, not a magic-string sentinel:
  - `kind: "any"` — items that have at least one name mapping (regardless of mapped value).
  - `kind: "matchesAny"` — items whose mapped name is in the `names` array (`names` is OR'd within the list). With `includeNoMapping: true`, items with no mapping rule also pass.
  - Omitting the `nameMapping` field entirely means "no filter on mapping" — different from `kind: "any"`.

---

## Flow 5: Review a journey

Look up how a journey is defined. Journeys are not yet exposed through the Avo MCP, so the agent falls back to matching metrics, which often capture the same intent.

Trigger prompts:
- "How is the signup flow defined?"
- "What's the checkout journey?"
- "Show me the onboarding journey"

Steps:

1. `search` for matching metrics. Use semantic mode if Smart Search is on (`query: "<user's phrasing>"`, `itemType: "metric"`). Otherwise use filter mode on `itemType: "metric"` with the closest name-based filters.
2. If one or more matching metrics are returned: present them with their event composition and properties, framed as the closest available answer to the user's question.
3. If no matching metrics are found: tell the user "I couldn't find any matching metrics for your question, and journeys are not available yet in the Avo MCP."

---

## Design intake gate (any request to create or design tracking)

A customer will rarely hand you a clean brief. Expect "create events for onboarding" or "add tracking for checkout" — and convert it, on your own initiative, into the full workflow below *before* you propose or create a single item. A short prompt is not permission to skip any of this.

1. **Bootstrap** the workspace (Flow 1).
2. **Read the rules** — `get { type: "workspaceConfig" }` — and state the casing, structure, and tense (and word choice, e.g. past tense, "clicked" vs "pressed") you will follow. Fall back to Avo's default only if none are configured, and say so.
3. **Recon with both lenses** — a semantic `search` (by meaning) **and** a structural filter `search` (by `categories` / `sources` / `properties`) — for items that already exist, and reuse them. The two lenses catch different things; running only one misses reusable items.
4. **Anchor a purpose** — if the prompt names no goal, ask (or state the assumption for) which metric or funnel the events serve. Design to the question the data must answer, not to the feature.
5. **Deliver** events (each in a category) **plus at least one metric you propose** (e.g. a conversion or drop-off funnel metric), tied to that purpose, in the workspace convention.

This skill covers the **existing-plan** case — follow **Flow 7** below; its steps elaborate each point. If recon shows the workspace is empty or near-empty, switch to the **data-designer-new-plan** skill (Flow 6, Kick off a new taxonomy).

---

## Flow 7: Design data for a product update

The big write flow. Takes a product requirement file or designs and produces a tracking plan branch. Use when the workspace already has a meaningful taxonomy and the user is adding to it; if the workspace is empty, use the **data-designer-new-plan** skill instead.

Trigger prompts:
- "Design tracking for this product requirement file"
- "Add tracking for the new onboarding flow"
- "Set up analytics for [feature]"
- "Improve our existing tracking for [area]"

Steps:

1. Analyze the context. If it includes designs (e.g. Figma frames), use them; otherwise rely on the text.
2. Check that the product update has clear problems, goals, and target metrics. If any of these are vague or missing, pause and ask the user:
   - "What problems is the team trying to solve with this product update?"
   - "What metrics are you hoping to move?"
   
   Wait for the user's response before continuing. Good tracking design starts from the question the data needs to answer, so this is a hard gate, not optional.
3. **Recon with both lenses.** Run a semantic `search` (by meaning, e.g. `query: "onboarding"`) **and** a structural filter `search` (by `categories` / `sources` / `properties`, no `query`) to see which events and properties already exist — the two lenses surface different items. `get` the promising hits. Reuse aggressively; never create a duplicate of an event or property that already exists. Recon is required even when the prompt is one line.
4. Read the workspace's tracking plan audit rules by calling `get` with `type: "workspaceConfig"` (which returns event naming, property naming, casing, and validation rules) to determine what naming convention to use for any new items you propose.
   
   When proposing new items, always look at what already exists in this workspace's tracking plan and match the patterns you find there. Specifically:
   - **For events.** Look at existing event names, especially in the same category or area. Match their structure (object-action vs action-object), casing, tense, and word choice. If the user is adding a step to an existing funnel, follow the start/step/complete pattern already in use.
   - **For properties on a specific event.** First look at the other properties already on that event. Match their casing, prefix style (e.g. `Product Name` / `Product ID` / `Product Category`), and how they describe state.
   - **For properties in general.** Also look at how the same kind of value is named elsewhere in the plan. If counts elsewhere use `X Count`, do not introduce `Number of X` on a new event.
   - **When multiple patterns exist.** Propose the most common one as the default and show the alternatives so the user can choose. Do not pick silently.
   
   Follow the rules in "Tracking design principles" below. See `examples/seed-taxonomy-reference.json` for naming, category, and funnel patterns to draw from when nothing comparable exists in the plan yet.
5. Return: a plan of what to reuse and what to create, every new event in a category, and **at least one metric you propose** (e.g. a conversion or drop-off funnel metric) tied to the problem the user named — propose the metric, don't just ask which one matters. Anchor every event back to the question it answers. Ask the user one of three things:
   - a) Do they have questions about decisions made?
   - b) Do they want to change something?
   - c) Should we proceed to create a branch?

Branch on the user's response:

- a) **Have questions** → return the reasoning behind the decisions made, then re-ask.
- b) **Want to change something** → take the change, update the plan, and re-ask.
- c) **Proceed to create a branch:**
   1. `workflow` with `action: create_branch` → returns `branchId`.
   2. `save_items` (batch) with the planned events, properties, and variants. Use `tempId` for forward references inside the batch.
   3. Return: a summary of what was done with a link to the branch. Recommend reviewing changes. Ask if they want to update the branch to "ready for review" and add reviewers.
   4. The user reviews in the Avo app and manually changes branch status and adds reviewers.

---

## Flow 8: Update a branch (small edits)

Sub-flow for non-breaking updates: filling in allowed values, adding categories, adding stakeholders, renaming, retyping, attaching properties to events.

Trigger prompts:
- "Create the missing allowed values for the screens I created"
- "Add categories and stakeholders for all the new items"
- "Rename `user_email` to `email`"

Steps:

1. If a branch is already in conversational context: use it.
2. If no branch is in context: ask the user whether to create a new branch or use an existing one.
   - If create: `workflow` with `action: create_branch` → returns `branchId`.
   - If use existing: `list_branches` or confirm the branch by name, then take that `branchId`.
3. `get` to resolve each target by name → `eventId` / `propertyId`. Never invent ids.
4. `save_items` with `op: update`, sending only the fields that change.

Context: this is the "non-editors making non-breaking changes" entry point, and it works equally well for editors making small fixes.

---

## Flow 9: Prioritize branches

Help the user pick which branch to tackle next.

Trigger prompts:
- "What branches am I assigned to review?"
- "What's on my plate in Avo?"

Steps:

1. `list_branches` filtered by status `ready for review` and reviewer set to the current user.
2. `get_branch_details` to read status and reviewers for each.
3. Return: the list of "ready for review" branches assigned to the user, ordered from oldest to newest by `creationDate` (the MCP does not expose a per-reviewer assignment timestamp; `creationDate` is the deterministic proxy for "has been waiting longest"). Ask which one they want to review.

---

## Flow 10: Review a branch

Walk through what's changing on a specific branch.

Trigger prompts:
- "Help me review the branch checkout-updates"
- "What's on branch X?"

Steps:

1. `list_branches` to resolve the branch name to a `branchId`.
2. `get_branch_implementation_guide` for the structured diff.
3. Return the structured diff. Ask if the user wants to see code snippets for any specific source (which is the entry point into Flow 11, Implement a branch).

Note: this flow will be updated once journey triggers are returned in the MCP. If the branch touches a journey today, the structured diff covers the underlying event and property changes.

---

## Flow 11: Implement a branch

Hand off branch changes into source code, then close the loop.

Trigger prompts:
- "Implement the update-signup-flow branch for iOS"
- "Get me the code changes for branch X on web"

Steps:

1. `list_branches` to confirm the `branchId`.
2. `get(type:"source")` to resolve `sourceId` — returns the workspace's sources as a markdown table; pick the matching `id` from the table.
3. If the user specified a source in their prompt: skip to step 5.
4. If no source was specified: return the list of sources and ask which source(s) they want the diff for. Wait for the user to specify.
5. `get_branch_code_snippets` for the requested source. One source at a time; this tool is expensive.
6. Agent implements the branch changes locally and commits them. **Do not push automatically.** Summarize what was committed (files touched, commit message) and ask the user to confirm before running `git push`. If the user prefers a patch-only path (no commit, no push), offer to leave the changes uncommitted in the working tree instead.
7. On user confirmation: push to git. Peer review happens in the customer's git workflow.
8. Once the git branch is merged: ask the user if they want to merge the Avo branch (only when all branches are fully implemented), or ask whether to push the source changes into a different branch to merge. Provide the link to the diff view of the branch.
9. The user clicks the link and merges from the Avo app.

Sub-prompt: "Are there branches I can merge now?" → return the list of branches that are ready to be merged and would otherwise leave git out of sync. Ask if the user wants to merge them.

---

## Flow 12: Analyze data (cross-MCP)

Multi-MCP rocket flow combining Avo with the user's data tool: any MCP that can query their actual event or warehouse data (Amplitude, Mixpanel, PostHog, BigQuery, Snowflake, Databricks, Redshift, and so on).

Trigger prompts:
- "Where are people dropping off in the signup funnel on mobile?"
- "How is the checkout funnel performing this week?"
- "Why are completion rates dropping for signup?"

Steps:

1. Use the Avo MCP to check whether the relevant funnel, event, or property is defined in Avo (Flow 2 or Flow 4 depending on phrasing).
2. Check whether a data MCP is connected (any MCP that can query the user's actual data: product analytics like Amplitude, Mixpanel, PostHog, or a warehouse like BigQuery, Snowflake, Databricks, Redshift).
3. **If a data MCP is connected:** use it to query the data behind the question.
4. **If no data MCP is connected:** ask the user what tool they use to query their event or warehouse data, and suggest they connect the corresponding MCP. Pause the flow there.
5. If the data returns gaps or anomalies that look like a tracking-plan issue: report what's missing. Example: "One of the signup events is missing the `Platform` property."
6. Suggest drafting an Avo branch to fix the gap. Example: "Add the `Platform` property to the [event]."
7. Wait for the user to approve the plan.
8. On approval: drop into Flow 7 at the "proceed to create a branch" step.

Keep phrasing tool-agnostic. Refer to "the connected data MCP" in user-facing messages, not a specific vendor, so the same flow works regardless of stack.

---

## Tool selection cheatsheet

When the user describes by **meaning** (a fuzzy concept, a feature, a goal): use `search` in semantic mode with `query`. This is Flow 2.

When the user names an **exact item** or pastes an id: use `get`. This is Flow 3.

When the user names a **structural relationship** ("events with property X", "metrics in category Y", "items mapped to Z"): use `search` in filter mode with no `query`. This is Flow 4.

When the user asks about a **branch by name**: use `list_branches` to resolve, then `get_branch_details` for status and reviewers, then `get_branch_implementation_guide` for the diff.

Prefer `get_branch_implementation_guide` over `get_branch_code_snippets` for review intent. Snippets are expensive and rarely what review-mode users want.

Default branch for `get` is `main`. When inspecting items on a non-main branch, pass the branch explicitly.

When the user (or a flow) needs the workspace's **audit rules** (naming convention, casing, validation rules) — for example before proposing new events or properties in Flow 6 or Flow 7 — use `get` with `type: "workspaceConfig"`. This is the canonical way to retrieve them; there is no separate audit-rules tool.

---

## Writing to a branch

Writes always happen on a branch. Never on main. The MCP never merges; merge stays in the Avo web app.

The first write tool call in a session triggers a scope escalation browser prompt (read → write). Warn the user once that they will see a re-auth prompt, then proceed.

Use `tempId` to cross-reference newly-created items inside a single `save_items` batch. A new property gets `tempId: "prop_new_1"`; a new event references it in its properties list by that same `tempId`. `tempId` only works within one batch; across batches, use the real id returned from the previous call.

Batch aggressively. One `save_items` call with twenty items is preferred over twenty calls.

`op: update` on a property modifies it everywhere it is used, not just on one event. If the user means "change this on one event only", they likely want an event-scoped override, not a property edit. Confirm before running destructive-sounding changes.

After any write, always link the user to the branch in the Avo web app and remind them that merge is a human step.

---

## Tracking design principles

Reference for Flow 7 (Design data for a product update). Use these as the design rules when proposing new events and properties to a user.

### Match existing patterns first

Before applying any abstract convention, look at what already exists in the plan. A workspace with a year of tracking has a real working convention encoded in its current names, and proposals that diverge from it create exactly the duplicates and inconsistencies the plan is supposed to prevent.

**For events.** Look at existing event names, especially in the same category or product area. Match their structure (object-action vs action-object), casing, tense, and word choice. If the user is adding a step to an existing funnel, follow the start/step/complete pattern already in use rather than inventing a parallel pattern.

**For properties on a specific event.** First look at the other properties already on that event. Match their casing, prefix style, and how they describe state. If existing properties on the event are `Product Name`, `Product ID`, `Product Category`, a new property should be `Product Color`, not `productColour` or `Color`.

**For properties in general.** Also look at how the same kind of value is named elsewhere in the plan. If counts elsewhere use `X Count`, do not introduce `Number of X` on a new event. If state snapshots elsewhere use the `Current X` prefix, follow that.

**When multiple patterns exist.** Propose the most common one as the default and surface the alternatives so the user can choose. Do not pick silently. Example: "I see both `Product ID` and `Product Id` used in the plan. `Product ID` appears on six events and `Product Id` on one; I'll go with `Product ID` unless you'd rather standardize the other way."

**Fall back to the abstract convention only when nothing comparable exists yet.** This is the case in Flow 6 (kick off a new taxonomy) or when entering a category that has no neighbors in the plan.

### Naming convention

Three dimensions matter, in this order:

1. **Structure.** Avo's recommended default is **object-action** (`Signup Completed`, `Cart Updated`, `Message Sent`). Action-object is also valid but less common in modern plans.
2. **Casing.** Avo's recommended default is **Title Case**. Other valid options: `snake_case`, `camelCase`, `PascalCase`, `lower case`, `kebab-case`. Pick one and stick to it. Some downstream destinations require specific casing, so check first.
3. **Tense.** Avo's recommended default is **past tense** (`Signup Completed`, not `Signup Complete`).

A naming convention may also include a set of allowed words. For example: "always use `game`, never `match`". When proposing names, prefer words the user has already used and flag synonyms for confirmation rather than silently substituting.

### Event design

**Use a start-milestone-complete pattern for any funnel.** This is the single highest-leverage choice in a tracking plan, because it makes drop-off visible without extra work. The seed taxonomy uses this pattern for Signup, Purchase, Subscription Upgrade, Subscription Downgrade, and Onboarding. Example: `Purchase Started`, `Purchase Step Completed`, `Purchase Completed`.

**Track actions, not UI elements.** `Signup Completed` is the action. `Complete Signup Button Clicked` is the UI. Names tied to UI elements rot the moment the button copy or layout changes. If button context matters, capture it as a property on the action event (`Button Copy`, `Button Location`).

**Push back when the user proposes an anti-pattern name.** Do not silently accept a name you would not propose yourself. When the user offers a vague or UI-coupled name like `Button Clicked`, `Event Happened`, `User Action`, or `Page Loaded`, say what is missing (which button, which action's outcome, on which surface) and propose a clearer alternative that follows the workspace's audit rules (retrieved via `get` with `type: "workspaceConfig"`). If the user insists, capture their reasoning in the event description so the choice survives the conversation.

**Build deep events, not many shallow ones.** A deep event has many properties that describe variations of the same user action; a shallow event would split each of those variations into its own thin event. The alternative (`Profile Picture Updated`, `User Name Updated`, `Birth Date Updated` as separate shallow events) fragments analysis and inflates the plan. Prefer one deep `Profile Configured` event with a `Profile Configuration Action` (Add / Update / Remove) and a `Profile Configured Item` (which field) — see `examples/seed-taxonomy-reference.json` for the worked example.

**An action with several outcomes is still one event — put the outcome in a property.** When a single user action can resolve more than one way — a payment that succeeds, is declined, needs extra verification, or is still processing; a setup step the user completes or skips — that is one event with an outcome property, not one event per outcome. Model `Payment Attempted` with a `Payment Outcome` of `Succeeded` / `Declined` / `Requires Authentication` / `Processing`, not separate `Payment Succeeded` and `Payment Failed` events. Splitting by outcome fragments the funnel and throws away the denominator: with one event you can ask "what share of payment attempts were declined?"; with two you've lost the attempt count, and every new outcome forces a new event. Same for a skippable step — model a `Step Outcome` (`Completed` / `Skipped`), not a separate skip event.

**Track only what matters.** Pre-release purpose meetings should produce a small set of events tied to the metrics the team committed to. Resist the urge to instrument everything. More events means more cost, more privacy surface, and more noise.

**Front-load events at the start and end of every important funnel.** The drop-off between the door and the conversion is where the insight lives. Always create both endpoints first, then fill in the milestones that matter.

### Property design

**Be explicit about what the property describes.** Do not name properties `Product` or `ID`; use `Product Name`, `Product ID`, `Product Category`, `Product Type`, `Product Price`, `Product SKU`. The seed taxonomy follows this pattern strictly.

**Pick a count format and reuse it.** Examples: `Product List Item Count`, `Current Product List Filter Count`. Or `Number of Passengers`. Or `Num Passengers`. Whichever you choose, stay consistent across the plan.

**Reuse an existing property before minting a near-duplicate.** Recon often turns up a property that already answers your question under a slightly different name — an existing `Search Result Count` when you were about to create `Suggestion Count`. Reuse it. Two properties measuring the same thing split analysis and are exactly the duplication recon exists to catch. Create a new one only when it genuinely measures something the existing property doesn't.

**Use `Current X` for at-the-time-of-event state** when there is a meaningful difference between in-the-moment state and persistent state. Example from the seed taxonomy: `Current Product List Filters`, `Current Product List Sorting`. This signals to the data consumer that this is an event-time snapshot.

**Do not share names between event and user properties.** They describe different things. Event properties are at-the-time-of-event values; user properties are latest known values. If both are useful, name the user property to indicate that (`Latest UTM Medium`).

**Three categories to think in:**
- **Event properties** describe the specific action (e.g. `Authentication Method`, `Game Mode`).
- **User properties** describe persistent state (e.g. `Subscription Status`, `User Name`).
- **System properties** are sent with every event and rarely change in-session (e.g. `Platform`, `App Version`, `Browser`).

**Use property constraints whenever the value space is finite.** Enums for `Authentication Method: ["Facebook", "Google", "Email"]`. Min and max for numeric values (`Search Result Position` minimum 1). Constraints prevent the "Android sends `facebook`, iOS sends `Facebook`, web sends `FB`" trap.

**Avoid Boolean properties for state that could plausibly grow past two values.** `Is Facebook Authenticated` will break the moment you add a third method. `Authentication Method` as a string scales. The seed taxonomy demonstrates this throughout: `Game Result` as a string instead of `Did Win` as a Boolean.

### Descriptions

**Every event and property gets a description.** No exceptions in proposals.

**Describe when the event fires — in language anyone can understand.** A good event description explains exactly *when* the event is sent, in plain words a product manager, a developer, and a marketer would all understand. Saying when it fires is the whole point: it tells a developer precisely when to send the event (so platforms stay consistent) and tells an analyst what the data means.

The enemy is **implementation vocabulary, not the word "when".** Say when the event fires in terms of what the *user* did, not how the system produced it. Drop the engineering framing — requests, responses, status codes, debounce, retries, "the client receives a successful response" — that's the part an analyst can't read and the most common reason a branch bounces in review.

- ❌ `Purchase Completed` — "Event sent when the client receives a successful response after the user completes a purchase." (The "when" is right; "client receives a successful response" is jargon only an engineer reads.)
- ✅ `Purchase Completed` — "Sent when a user successfully pays and their order is placed." (Says exactly when it fires — and a developer, analyst, and marketer all understand it.)

If two platforms could fire on subtly different moments, disambiguate in plain language ("…when the order is confirmed, not when Pay is tapped"), not by describing the network call.

**Property descriptions clarify the value, not just the name.** "Which search result the user selected. 1 for the top search result, 2 for the one below, and so on."

**Write descriptions (and names) for the data consumer, not the implementation** — see the principle below.

### Write for the data consumer, not the implementation

Names and descriptions are read by analysts, PMs, and other data consumers who don't know — and shouldn't need to know — how the feature is built. Name and describe every item in the product's user-facing vocabulary, never the implementation's.

This is the single most common failure when an AI designs tracking from a spec or codebase: the jargon lives in the code, and the AI forwards it straight into names and descriptions. Watch for four kinds of leaked jargon:

- **Algorithm / model names** — the ranking method, scoring formula, or library name.
- **Internal field or column names** — a code identifier or DB column lifted straight into a property name.
- **Engineering status enums** — raw backend state values (e.g. `DEGRADED`, `FALLBACK`) copied verbatim.
- **Statistics / ops shorthand** — `p90`, `stddev`, "distribution", raw unit names.

A data consumer who meets one of these opens the item and asks "what does this word mean?" — exactly the review feedback that sends a branch back.

Example:

- ❌ `Ranker Confidence Logit` — "Normalized logit from the v2 scoring model."
- ✅ `Recommendation Strength` — "How strong a match this recommendation is for the user. Higher means a better fit."

Apply this:

- **Translate, don't transplant.** Map each internal concept to how a non-engineer would say it. Strip algorithm names, internal identifiers, status codes, and stats shorthand from names.
- **Describe the meaning, not the formula.** Say what the value tells the reader and which direction is good, not how it's computed.
- **Re-read as a stranger before saving.** Before `save_items`, read each name and description as someone with zero implementation knowledge. If a word only makes sense to someone who's read the code, rewrite it.

### A note on Boolean properties

Booleans are a trap in tracking plans. They feel natural for binary-looking actions, but most product state expands over time. Prefer strings with constraints; you can always add new allowed values without breaking existing data.

Renaming a Boolean to a two-value enum does not escape the trap — an enum whose only values are `Yes`/`No` (or `True`/`False`, `On`/`Off`) is a Boolean in disguise and inherits the same problem. The fix is to name the *dimension* the value lives on with values that carry product meaning: not `Requires Authentication: Yes / No`, but a `Payment Outcome` where `Requires Authentication` sits alongside the other ways a payment resolves. If a dimension genuinely only ever has two product-meaningful states, a two-value enum is fine — just make the values describe the states, not answer yes/no.

### Quick checklist when proposing an event

- Action-based name, not UI-based.
- Matches the pattern of neighboring events in the same category. (Local first, abstract convention second.)
- Has a description that says when the event fires in plain language — no engineering jargon — so a product manager, developer, and marketer all understand it.
- Properties cover the question the data needs to answer.
- Property names match neighboring properties on the same event and across the plan.
- Property names are explicit, not ambiguous.
- Names and descriptions use the data consumer's vocabulary, not the implementation's (no algorithm/model names, internal field names, status enums, or stats shorthand).
- Constraints applied where the value space is finite.
- No Booleans for state that might grow.
- Belongs to a category that matches a funnel or product area.
- Either at the start, the end, or a meaningful milestone of a funnel. Not both random and orphaned.

---

## Cross-cutting policies

Always run Flow 1 (Start session) before any other workspace-scoped call. Every workspace-scoped tool needs `workspaceId`.

Resolve names to ids before writing. Never invent ids, branch names, or property names. If you have a name and not an id, call `get` or `search` first.

`search` has two modes in a single tool. Pass `query` for semantic mode (Flow 2). Omit `query` and pass structural filters for filter listing (Flow 4). Pick the mode from the user's phrasing.

Semantic search requires Smart Search to be enabled in the workspace. Detect this on first failure and fall back to exact-name `get`, with a one-line nudge about enabling Smart Search.

When the user asks about journeys today, fall back to matching metrics per Flow 5 and explicitly tell them journeys are not available yet in the MCP.

Keep cross-MCP phrasing vendor-agnostic. Use "the connected data MCP" in user-facing messages.

---

## Gotchas

Agents skip `list_workspaces` on the first prompt and pass an invented id. Always run Flow 1 first.

Agents over-call `get_branch_code_snippets` when the user asked "what's on the branch." Prefer `get_branch_implementation_guide` for review intent.

Agents try to call write tools on the `read` scope and get an auth error mid-conversation. Warn the user about the scope escalation prompt before calling the first write tool, not after the error.

Agents create duplicate events because they skipped the recon in Flow 7. Always `search` or `get` to check for existing items before proposing new ones.

Agents skip the problems / metrics gate in Flow 7 because it feels like a delay. It is not a delay; it is the only way to design tracking that answers a real question. Treat it as required.

Agents invent new naming patterns instead of matching what already exists in the workspace. Always look at neighboring events and properties first; fall back to the abstract convention only when nothing comparable exists.

Agents propose vendor-specific commands in Flow 12 ("run this Amplitude query") instead of asking the user what data MCP they have. Ask first, then act.

Agents name and describe events/properties after the implementation — algorithm/model names, internal field names, raw status enums, stats shorthand — instead of the data consumer's vocabulary. Translate every internal concept into product language and re-read each name/description as a stranger before saving (see "Write for the data consumer, not the implementation").

`save_items` returns ids you need for future cross-batch references. Capture them in conversation state.

The first call to a write tool of a session opens a browser. On a headless or automated setup this fails; surface the error rather than retrying.

---

## Reference

See `examples/seed-taxonomy-reference.json` for a slim, curated reference taxonomy showing every canonical pattern (eight events, two metrics, four categories). Use it as the default reference for design-data flows when nothing comparable exists in the plan yet.

For the canonical, always-current tool list, see the Avo Tools reference: https://www.avo.app/docs/reference/avo-mcp/tools
