---
name: avo-mcp-new-tracking-plan
description: Bootstrap a brand-new Avo tracking plan from scratch through the Avo MCP. Use when the workspace is empty or near-empty and the team is deciding what to track — runs a purpose meeting (problems, goals, metrics, key funnels), proposes a naming convention (structure/casing/tense), a starting set of categories, and start-milestone-complete events with event/user/system properties and constraints, then creates a branch. Triggers on "we're new to Avo", "where do we start", "set up tracking from scratch", "what should we track", "bootstrap a tracking plan", "design our first tracking plan". If the workspace already has a substantial plan, use the avo-mcp skill instead.
---

# Avo MCP — Kick Off a New Tracking Plan

This skill teaches the agent how to bootstrap a brand-new Avo tracking plan from scratch through the Avo MCP server: confirm the workspace is empty, run a purpose meeting, agree a naming convention, and seed the first categories, events, properties, and metrics — then create a branch.

For working with an *existing*, substantial tracking plan — exploring and searching, looking items up, designing tracking against what's already there, reviewing and implementing branches, cross-MCP analysis — use the **avo-mcp** skill instead. Quick empty check: run `search` with `itemType: "event"` and no `query`; if it returns very few results, you're greenfield and this skill applies. If the plan is substantial, switch to **avo-mcp**.

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

## Design intake gate (any request to create or design tracking)

A customer will rarely hand you a clean brief. Expect "create events for onboarding" or "add tracking for checkout" — and convert it, on your own initiative, into the full workflow below *before* you propose or create a single item. A short prompt is not permission to skip any of this.

1. **Bootstrap** the workspace (Flow 1).
2. **Read the rules** — `get { type: "workspaceConfig" }` — and state the casing, structure, and tense (and word choice, e.g. past tense, "clicked" vs "pressed") you will follow. Fall back to Avo's default only if none are configured, and say so.
3. **Recon with both lenses** — a semantic `search` (by meaning) **and** a structural filter `search` (by `categories` / `sources` / `properties`) — for items that already exist, and reuse them. The two lenses catch different things; running only one misses reusable items.
4. **Anchor a purpose** — if the prompt names no goal, ask (or state the assumption for) which metric or funnel the events serve. Design to the question the data must answer, not to the feature.
5. **Deliver** events (each in a category) **plus at least one metric you propose** (e.g. a conversion or drop-off funnel metric), tied to that purpose, in the workspace convention.

This skill covers the **empty / near-empty** case — follow **Flow 6** below. If recon shows the plan is already substantial, switch to the **avo-mcp** skill (Flow 7, Design data for a product update).

---

## Flow 6: Kick off a new taxonomy

For a brand-new workspace or one with no real tracking plan yet. The point is to bootstrap problems and metrics first, then a naming convention, then a starting set of categories and events. Do not jump straight into creating events.

Trigger prompts:
- "We're new to Avo, where do we start?"
- "Help us design our first tracking plan"
- "Set up tracking from scratch"
- "What should we track?"
- "Bootstrap a tracking plan for our product"

Steps:

1. Confirm the workspace is empty or near-empty. Run `search` with `itemType: "event"` and no `query` (filter listing mode); if it returns very few results, treat this as a fresh start. If the plan is substantial, redirect to the **avo-mcp** skill (Flow 7, Design data for a product update).
2. Ask the user about the product. What does the product do, who are the users, and what stage of growth is it at? Keep this short; one or two sentences is enough to anchor the rest.
3. Run the purpose meeting checklist with the user before designing anything:
   - "What topics does this release cover?"
   - "What problems is the team trying to solve?"
   - "What goals and metrics will tell us we're succeeding?"
   - "Which one or two user funnels matter most right now (signup, checkout, onboarding, etc.)?"
   
   Wait for the user's answers before proposing structure. If answers are vague, push back: good tracking design starts from the question the data needs to answer.
4. Propose a naming convention. Default to Avo's recommendation unless the user already has a convention or their analytics destination demands otherwise:
   - Structure: object-action (e.g. `Signup Completed`)
   - Casing: Title Case
   - Tense: past tense
   
   Show the user one or two example events in their proposed convention, name the three dimensions explicitly (structure, casing, tense), and ask them to confirm or adjust. Capture the chosen convention so subsequent steps stay consistent.
5. Propose a starting set of categories that mirror the funnels the user named. Pick a small subset (three to six) that match this product. Examples from the seed reference taxonomy: `Authentication`, `Purchase`, `Navigation`, `Search`. The full seed taxonomy adds `Onboarding`, `Cart`, `Wishlist`, `Subscription`, `Workspace`, `Gameplay` for breadth.
6. For the highest-priority funnel, propose a starting set of events using the start-milestone-complete pattern from the seed taxonomy. Examples:
   - `Signup Started`, `Signup Step Completed`, `Signup Completed`
   - `Purchase Started`, `Purchase Step Completed`, `Purchase Completed`
   - `Onboarding Started`, `Onboarding Step Completed`, `Onboarding Completed`
   
   This makes drop-off visible by design.
7. For each event, propose properties in three categories:
   - **Event properties** specific to the action (e.g. `Authentication Method` on `Signup Completed`).
   - **User properties** describing state that persists across events (e.g. `Email`, `Subscription Status`).
   - **System properties** sent with all events (e.g. `Platform`, `App Version`).
8. Read the workspace's tracking plan audit rules by calling `get` with `type: "workspaceConfig"` (which returns event naming, property naming, casing, and validation rules), and adjust proposed names to match.
9. Return a complete proposal: convention, categories, events with descriptions, properties with descriptions and constraints, and **at least one metric you propose** (a conversion or drop-off funnel metric) tied to the funnel the user named. Anchor every event back to the problem or metric it serves. Ask the user one of:
   - a) Do they have questions about decisions made?
   - b) Do they want to change something?
   - c) Should we proceed to create a branch?
10. Branch on the user's response:
    - a) **Have questions** → return the reasoning behind the decisions made, then re-ask.
    - b) **Want to change something** → take the change, update the plan, and re-ask.
    - c) **Proceed to create a branch:**
       1. `workflow` with `action: create_branch` → returns `branchId`.
       2. `save_items` (batch) with the planned categories, events, properties, and metrics. Use `tempId` for forward references inside the batch (a new property gets `tempId: "prop_new_1"`; a new event references it by that same `tempId`). Batch aggressively — one call with many items beats many calls.
       3. Return: a summary of what was done with a link to the branch. Recommend reviewing changes. Ask if they want to update the branch to "ready for review" and add reviewers.
       4. The user reviews in the Avo app and manually changes branch status and adds reviewers. Merge is always a human step in the Avo web app.

Pointers:
- See "Tracking design principles" below for the naming, event, and property guidance the proposals should follow.
- See `examples/seed-taxonomy-reference.json` for a worked reference taxonomy showing every canonical pattern (start-step-complete funnels, deep events, property naming, constraints, conversion and segmentation metrics, categories).

---

## Tool selection cheatsheet

When the user describes by **meaning** (a fuzzy concept, a feature, a goal): use `search` in semantic mode with `query`.

When the user names an **exact item** or pastes an id: use `get`.

When the user names a **structural relationship** ("events with property X", "metrics in category Y", "items mapped to Z"): use `search` in filter mode with no `query`.

When the user asks about a **branch by name**: use `list_branches` to resolve, then `get_branch_details` for status and reviewers, then `get_branch_implementation_guide` for the diff.

Default branch for `get` is `main`. When inspecting items on a non-main branch, pass the branch explicitly.

When you (or a flow) need the workspace's **audit rules** (naming convention, casing, validation rules) — for example before proposing new events or properties — use `get` with `type: "workspaceConfig"`. This is the canonical way to retrieve them; there is no separate audit-rules tool.

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

Reference for Flow 6 (Kick off a new taxonomy). Use these as the design rules when proposing new events and properties to a user.

### Match existing patterns first

Before applying any abstract convention, look at what already exists in the plan. A workspace with a year of tracking has a real working convention encoded in its current names, and proposals that diverge from it create exactly the duplicates and inconsistencies the plan is supposed to prevent. (In a brand-new workspace there are few or no neighbors yet, so you will fall back to the abstract convention more often — but still check for the handful of items that may already exist.)

**For events.** Look at existing event names, especially in the same category or product area. Match their structure (object-action vs action-object), casing, tense, and word choice. If the user is adding a step to an existing funnel, follow the start/step/complete pattern already in use rather than inventing a parallel pattern.

**For properties on a specific event.** First look at the other properties already on that event. Match their casing, prefix style, and how they describe state. If existing properties on the event are `Product Name`, `Product ID`, `Product Category`, a new property should be `Product Color`, not `productColour` or `Color`.

**For properties in general.** Also look at how the same kind of value is named elsewhere in the plan. If counts elsewhere use `X Count`, do not introduce `Number of X` on a new event. If state snapshots elsewhere use the `Current X` prefix, follow that.

**When multiple patterns exist.** Propose the most common one as the default and surface the alternatives so the user can choose. Do not pick silently. Example: "I see both `Product ID` and `Product Id` used in the plan. `Product ID` appears on six events and `Product Id` on one; I'll go with `Product ID` unless you'd rather standardize the other way."

**Fall back to the abstract convention only when nothing comparable exists yet.** This is the common case in a new taxonomy or when entering a category that has no neighbors in the plan.

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

`search` has two modes in a single tool. Pass `query` for semantic mode. Omit `query` and pass structural filters for filter listing. Pick the mode from the user's phrasing.

Semantic search requires Smart Search to be enabled in the workspace. Detect this on first failure and fall back to exact-name `get`, with a one-line nudge about enabling Smart Search.

---

## Gotchas

Agents skip `list_workspaces` on the first prompt and pass an invented id. Always run Flow 1 first.

Agents skip the purpose meeting checklist in Flow 6 and start designing immediately. Do not. A new taxonomy without anchored problems and metrics will rot inside three months.

Agents invent new naming patterns instead of matching what already exists in the workspace. Even in a fresh workspace, check for the handful of items that may exist; otherwise fall back to the abstract convention.

Agents create duplicate events because they skipped the recon. Always `search` or `get` to check for existing items before proposing new ones.

Agents name and describe events/properties after the implementation — algorithm/model names, internal field names, raw status enums, stats shorthand — instead of the data consumer's vocabulary. Translate every internal concept into product language and re-read each name/description as a stranger before saving (see "Write for the data consumer, not the implementation").

Agents try to call write tools on the `read` scope and get an auth error mid-conversation. Warn the user about the scope escalation prompt before calling the first write tool, not after the error.

`save_items` returns ids you need for future cross-batch references. Capture them in conversation state.

The first call to a write tool of a session opens a browser. On a headless or automated setup this fails; surface the error rather than retrying.

---

## Reference

See `examples/seed-taxonomy-reference.json` for a slim, curated reference taxonomy showing every canonical pattern (eight events, two metrics, four categories). Use it as the default reference for new-taxonomy flows.

For the canonical, always-current tool list, see the Avo Tools reference: https://www.avo.app/docs/reference/avo-mcp/tools
