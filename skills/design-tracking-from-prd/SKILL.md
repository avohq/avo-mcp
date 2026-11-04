---
name: design-tracking-from-prd
description: Turn a PRD, product spec, Figma file, screenshots, or a feature brief into tracking design in an existing Avo tracking plan through the Avo MCP. Produces a reviewable design proposal (events and properties to reuse or create, at least one metric, and where each event fires), then — on approval — an Avo branch. Triggers on "here's our PRD, design the tracking", "what should we track for this feature", "turn this spec / Figma into events", "draft the tracking spec for this ticket". If the workspace is empty use data-designer-new-plan; to import a plan that already exists elsewhere use import-tracking-plan.
---

# Design tracking from a PRD

A product manager drops in a PRD, a Figma link, or a few paragraphs about a feature. This skill turns that into a tracking design that fits the existing plan — reusing what's there, following the workspace's conventions, tied to the question the data must answer — and writes it to an Avo branch once the user approves.

**What you produce:**

1. A **design proposal** the team can review: reuse vs. create, every new event in a category, at least one metric, and where each event fires.
2. After approval, an **Avo branch** with the changes. The MCP never merges; merging is a human step in the Avo web app.

This skill is the PRD-shaped entry point to the `data-designer` skill's **Flow 7 (Design data for a product update)**. Follow its **Design intake gate** and **Tracking design principles** in full; the steps below add what is specific to working from a document.

---

## Step 1: Read the document for behaviour, not features

From the PRD, Figma frames, or screenshots, extract:

- **The user actions** the feature introduces or changes — what the user *does*, not which button they press.
- **The questions the team wants answered** — success metrics, goals, "we'll know this works when…". PRDs often state these; quote them back.
- **The screens or steps** in order, if the document has them. You'll use these to say where each event fires.
- **Anything platform-specific** (web only, mobile only) that maps to Avo sources.

If the document is missing the problem, a goal, or a target metric, stop and ask: "What problem does this solve, and which metric should move?" This gate is not optional; designing without it produces events nobody uses.

## Step 2: Load the rules

1. `list_workspaces` and set the workspace (data-designer Flow 1).
2. `get` with `type: "workspace_config"` — naming format, casing, allowed words, and which rules are Enforce.
3. **Team rules.** If the user or the repo provides a tracking style guide (a rules file, a Confluence page, a glossary), read it and treat it as more specific than Avo's defaults. The workspace's Enforce rules still win, because they block the branch at review. Say which rules you're following.

## Step 3: Recon

Run both lenses for every action from Step 1 (data-designer Flow 7, step 3): a semantic `search` by meaning and a structural filter `search` by category, source, or property. `get` the promising hits. Semantic hits can live on other people's unmerged branches: only reuse items whose `Branch` is `master` (main). Look at neighbouring events in the same category and match their structure, casing, tense, and property naming. If the workspace has an allowed-words list for parts of event names, every new name must use those words; if none fits, raise it as an open question instead of breaking the rule.

**Gateway workspaces.** If `search` with `itemType: "gateway"` returns any gateways, check how a neighbouring event reaches its destinations: `get` it with `checkpoint: {kind: "destination", destinationId}`. Look at the event and its properties: renames (e.g. to snake_case for a warehouse), properties the gateway adds, and anything dropped by an allow-list. A new event and its new properties need the same treatment to arrive consistently, and a new event or property that isn't on an output's allow-list won't arrive at all. Add a **Routing** section to the proposal listing, per destination, the event and property renames and allow-list additions it needs.

If the feature extends an existing journey, `search` with `itemType: "journey"` and walk the relevant one with `get` (`type: "journey"`) to see which events already fire on those screens.

## Step 4: Propose

Present one proposal, in this shape:

```
Tracking proposal — <feature name>
Question it answers: <the metric or question from the PRD>

Reuse (already in the plan)
  • Checkout Started — add property "Entry Point" (exists on 4 other events)

Create
  • Express Checkout Completed  [Checkout]
      Fires when: the user confirms payment with a saved card and the order is placed.
      Properties: Payment Method (reuse), Order Value (reuse), Express Checkout Source (new: "Cart", "Product Page")

Metric
  • Express checkout conversion: Checkout Started → Express Checkout Completed, by Entry Point

Where it fires
  • Cart screen → "Pay with saved card" → Express Checkout Completed

Open questions
  1. Does express checkout exist on Android at launch, or iOS and web only?
```

**New properties on events that already ship.** Adding a property to a live event makes it "always sent" by default, so every existing call site needs updating before the code compiles or validates. Say so in the proposal, and offer to mark it "sometimes sent" on that event for a transition (`eventConfigs` with `setPresence`) if older app versions won't send it.

Then ask: questions about the decisions, changes, or go ahead and create the branch?

## Step 5: Write the branch

On approval, follow data-designer Flow 7 (c):

1. `workflow` with `action: "create_branch"`. Use the feature name in the branch name and link the PRD in the description.
2. `describe_tool(tool: "save_items", type: "<type>", op: "<op>")` before the first write of each type and op: `create` for new items, `update` for reused ones you're changing (for example attaching a new property to an existing event). The two ops take different fields. Each item has the shape `{op, type, id?, name?, tempId?, fields}`, with type-specific fields inside `fields`.
3. One batched `save_items` call where possible: new properties with `tempId`s, events referencing `$tmp:<tempId>`, then categories in a follow-up batch (categories can't attach to items created in the same batch).
4. **Where it fires.** If you have screenshot URLs, seed each new event's `triggers` on create (`{sources, description, image: {source: "url", url}}`), with the description saying what the user does on that screen. Journeys themselves can't be written through the MCP: if the team uses journeys, give them the per-screen trigger list so they can add it in the journey builder.
5. Read back the branch with `get` (`type: "branch"`, `include: ["all_changes"]`) and confirm it matches the approved proposal.
6. Return the branch link and recommend the next step: review in Avo, or run the review-tracking-branch skill before asking humans to review.

---

## Gotchas

- **Agents design to the feature list instead of the question.** Every event must trace back to a metric or question from Step 1.
- **Agents copy the PRD's or the code's vocabulary into names.** PRDs are full of internal project names and component names. Translate them into what a data consumer would call the action (data-designer: "Write for the data consumer, not the implementation").
- **One event per button.** If the PRD lists five buttons that do the same action with different context, that's one event with a property.
- **Outcomes as separate events.** A payment that can succeed or fail is one event with an outcome property, not two events.
- **Updating a branch that's already in review** is rejected unless the call passes `onReviewedBranch: {kind: "revertToDraft"}`, which sends it back to Draft. Ask the user first.
- **Promising journey writes.** The MCP reads journeys but can't create or edit them. Say so; don't imply the branch updated a journey.
