---
name: branch-implementation-handoff
description: Turn an Avo branch into an implementation handoff for developers through the Avo MCP — a shareable page with every changed event, where it fires (journey screens, trigger markers, property conditions), the full payload, and the per-platform code diff; and optionally a ready-to-file ticket in Jira, Linear, or GitHub Issues. Read-only on Avo. Triggers on "create a Jira ticket for this branch", "hand this branch off to the developers", "write the implementation ticket", "make a dev spec for branch X", "share the tracking spec with IT". To implement the branch in code yourself, use the data-designer skill's Implement a branch flow instead.
---

# Branch → implementation handoff

When a tracking branch is ready, someone has to tell developers exactly what to build: which events, fired where, with which values, in which code. Teams do this by hand with screenshots and copy-pasted specs. This skill builds the handoff from the branch in a couple of minutes.

**What you produce:**

1. **A handoff page** (HTML or Markdown, whatever the user's client can share): per changed event, where it fires, the full payload, and the code diff for each platform, never truncated.
2. **Optionally, an implementation ticket** in the user's tracker (Jira, Linear, GitHub Issues) via that tracker's MCP, with the same content in ticket form and a per-platform checklist.

This skill only reads from Avo. It never writes to the tracking plan.

---

## Step 1: Resolve the branch

1. `list_workspaces` and set the workspace.
2. Resolve the branch: a branch link contains the id (`.../schemas/<workspaceId>/branches/<branchId>/...`); otherwise use `list_branches` with `branchName`. Get the branch's **name** from `list_branches` too: the branch overview is headed by its id, not its name.
3. `get` with `type: "branch"`, `branchId`, `include: ["overview", "all_changes"]`. This tells you **which** events and properties changed and the kind of change (`changed: properties`, `changed: allowed values`), plus review status and impacted sources (as ids).
4. Resolve the impacted source ids to names and languages with `search` (`itemType: "source"`). If the overview says "Impacted sources: none" but events changed, use the sources listed on each changed event instead.
5. **Work out what exactly changed.** The diff doesn't list the added property, the added value, or new triggers. For each modified item, `get` it twice — on the branch (`branchId`) and on main (no branch) — and compare: properties added or removed, allowed values added or removed, triggers added or removed. New items only need the branch read.
6. Check whether the workspace routes events through gateways: `search` with `itemType: "gateway"`. If it returns any, Step 3b applies.

If the branch isn't approved yet, or main has moved ahead of it, say so in the handoff header. Developers should know whether the spec might still change.

## Step 2: Collect where each event fires

For each new or modified event:

1. Use the branch read of the event from Step 1 (`get`, `type: "event"`, `branchId`). Its **Triggers** section is the source of truth for where the event fires.
2. For each trigger, keep: trigger name, journey, screen, screenshot URL, marker (`x`, `y`, `w`, `h`), and property conditions (e.g. `page_name is "Cart details"`). Standalone triggers have a description and maybe a screenshot.
3. Real trigger data is often incomplete. Handle it explicitly:
   - **No screen:** list the trigger by name and mark it "screen not documented".
   - **A screen but no screenshot:** name the screen and mark it "no screenshot".
   - **Several triggers with the same name:** keep them apart by journey and screen; they usually fire on different screens.
   - **A bare trigger line with no id and no fields:** a label, not a trigger. Skip it.
   - **A condition you can't resolve** (e.g. the value to send isn't stated): say what you inferred and label it as an inference.
4. If the event has no triggers at all, say so ("No trigger documented — confirm with the tracking plan owner") rather than guessing the firing moment from its name.

For each **removed** event, `get` it on main (no `branchId`) for its name and where it fired today, so developers can find the call sites. Its handoff section says what to delete; it needs no payload table.

Don't walk whole journeys for this: a journey read shows each screen's own events, not the triggers of the event you're handing off. Use it only if the user asks for the surrounding flow.

## Step 3: Collect the payload

The payload table lists **every** property the event sends, not only what changed: developers implement the whole call. For each property on those events, `get` with `type: "property"` and `branchId`. Record type, presence, list or not, allowed values, and **event-specific allowed values** (a property may allow fewer values on this event than globally). The event read lists the event's properties but not their allowed values, so this property read is required.

For object properties, the child fields aren't in the property read. Take them from the code snippet of a codegen source (Step 4), or `get` each child property by name.

Property bundles attached to the event are listed by name only. `get` each bundle (`type: "property_bundle"`) and include its properties in the payload; they're sent like any other property.

Build one payload table per event, marking what's new on this branch.

## Step 3b: What each destination receives (gateway workspaces)

When events pass through a gateway, what the code sends and what a destination receives can differ: outputs rename events and properties, add properties, and drop anything not on their allow-list. Developers and the people who asked for the change both need to know.

1. For each new or modified event, take the destinations it routes to (the event's **Sources → destinations** lines), and resolve their ids with `search` (`itemType: "destination"`).
2. For each destination that matters to the change (the branch description usually says, e.g. "reaches Meta for retargeting"; otherwise ask), `get` the event with `branchId` and `checkpoint: {kind: "destination", destinationId}`.
3. The result is the event as that destination receives it: the name it arrives under, properties added, renamed ("sent as"), and **dropped by an allow-list**, plus whether it reaches the destination at all.
4. Add a "What each destination receives" table to the handoff. **If something this branch adds is dropped before a destination that needs it** (a new property removed by the output's allow-list), put that at the top of the handoff as a problem to fix before implementation, not a footnote.

## Step 4: Collect the code

1. Use the impacted sources resolved in Step 1. Sources marked "not configured" have no language and no code snippets.
2. For each configured source the user cares about (ask if there are more than three), `get` with `type: "branch"`, `branchId`, `include: ["code_snippets"]`, and `sourceId`. One source per call: this call is expensive.
3. The output is already a per-event diff against main (`+` added, `-` removed lines), with `avo pull` or manual-removal guidance per event, including removed events. Keep it **verbatim and complete**. Each event in the snippet output is tagged `[Codegen]` (exact generated code) or `[Pseudocode]` (illustrative, for manually instrumented sources); carry that label into the handoff.

## Step 5: Build the handoff page

Structure:

```
# Implement tracking: <branch name>
Status: <review status> · Avo branch: <link> · Sources: Web, iOS, Android

## Summary
One line per event: new / modified / removed, and what changed.

## <EVENT NAME> (removed)
Where it fired on main, and which calls to delete per source.

## <EVENT NAME> (modified)
### Where it fires
For each trigger: screen name, the screenshot with the trigger area highlighted,
what the user does, and the property condition ("page_name is \"Cart details\"").
### Payload
| Property | Type | Presence | Allowed values on this event | New? |
### Code
One block per source: the code diff from Step 4, verbatim.
### What each destination receives (gateway workspaces only)
| Destination | Arrives as | Renamed | Added by gateway | Dropped |

## Checklist
- [ ] Web (TypeScript): run `avo pull`, update call sites
- [ ] iOS (Swift): …
## Acceptance
- [ ] <event> sends <property> on every platform
- [ ] Values match the screen it fired from
```

**Trigger markers:** `x`, `y`, `w`, `h` are fractions of the screenshot's width and height (0 to 1). Draw the highlight yourself: in HTML, an absolutely positioned box over the image at `left: x*100%`, `top: y*100%`, `width: w*100%`, `height: h*100%`. When `w` and `h` are missing, the marker is a point; draw a dot.

**Descriptions:** some descriptions can come back as raw rich-text JSON (`{"ops":[{"insert":"…"}]}`). Use the `insert` text, never the JSON.

Present the page to the user before sharing or filing anything.

## Step 6: File the ticket (optional)

1. Check which tracker MCP is connected (Jira, Linear, GitHub). If none, give the ticket body as Markdown for the user to paste.
2. Draft the ticket from the page: summary, where it fires (with screenshots), payload table, code per platform, checklist, acceptance criteria, and a link back to the Avo branch. Link the handoff page only if it has a URL (for example a published page); otherwise put its content in the ticket. Never invent a link.
3. Ask whether they want one ticket for the branch or one per platform. Show the draft and **create it only after the user confirms.**
4. Screenshots: some trackers don't render external image URLs. Upload the images if the tracker's MCP supports it; otherwise link them.

---

## Gotchas

- **Truncated code.** The point of the handoff is that nothing is cut off. Never abbreviate a snippet or a payload table.
- **Guessing where an event fires.** Use the triggers. If there are none, say so.
- **Missing per-event allowed values.** The global allowed values on a property aren't the whole story; check the event-specific restrictions from `get` on the property.
- **Calling `code_snippets` for every source by default.** Ask which platforms matter; each call is expensive.
- **Writing to Avo.** This skill is read-only. If the spec is wrong, tell the user to fix the branch first and regenerate.
