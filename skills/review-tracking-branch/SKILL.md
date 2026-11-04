---
name: review-tracking-branch
description: Review an Avo tracking-plan branch with an agent before (or alongside) human reviewers, through the Avo MCP. Checks every change against the workspace's audit rules, the team's own design rules, existing naming patterns, duplicates, description quality, PII, and whether each event says where it fires; produces findings per item (blocker, suggestion, question) and a readable summary for reviewers outside Avo. Also lists the branches waiting for the user's review. Triggers on "review branch X", "check this branch before I send it for review", "is this branch ready", "lint our tracking changes", "what's waiting for my review in Avo". Read-only: never approves or merges.
---

# Review a tracking branch

Human review of a tracking branch catches naming slips, duplicates, vague descriptions, and missing context, but it's slow and reviewers outside the data team struggle to read the diff. This skill does the first pass: it checks every change against the rules and the existing plan, and produces findings a human can act on in minutes.

**What you produce:**

1. **Findings per changed item**, each marked **Blocker**, **Suggestion**, or **Question**, with the rule or evidence behind it and a concrete fix.
2. **A readable summary** of the branch for reviewers who don't live in Avo: what changes, why, and where each event fires. For a full visual spec with screenshots and code, hand off to the branch-implementation-handoff skill.

This skill is read-only. It never approves, merges, or edits the branch. The user decides what to fix.

---

## Mode A: What's waiting for my review?

1. `list_workspaces` and set the workspace.
2. `list_branches` with `branchStatuses: ["ReadyForReview"]` and `reviewerEmail` set to the user's email (ask for it if you don't know it; it isn't filled in automatically). Results come newest first and paginated: follow `nextPageToken` to the end, then sort oldest first yourself.
3. List every branch from the `list_branches` summary alone: name, creator, age. Don't open each one; a reviewer can have dozens.
4. If the list is long, flag likely-abandoned branches (for example, waiting months with no activity) separately and suggest the user closes or reassigns them.
5. Ask which branches to look at, then `get` with `type: "branch"`, `include: ["overview"]` for those only, and report size and whether main has moved ahead of the branch. A branch behind main should pull main before a meaningful review; tell the user, and don't run `pull_main` yourself.

The overview shows how many approvals a branch has, not who approved. Don't tell the user a specific reviewer has approved.

## Mode B: Review one branch

### Step 1: Load the branch and the rules

1. Resolve the branch (from a link `.../branches/<branchId>/...` or `list_branches` with `branchName`).
2. `get` with `type: "branch"`, `include: ["overview", "all_changes", "audit"]`. The audit lists the issues the branch adds compared with main, each with its severity, whether it blocks the branch, and its rule key. On a plan that doesn't list audit issues you only get counts; then check the rules by hand.
3. `get` with `type: "workspace_config"`: naming format, casing, allowed words, and each rule's severity. **Enforce** rules block the branch at review; **Warning** rules don't. Use it for what the audit doesn't list.
4. **Team rules.** If the user or repo has a tracking style guide or glossary, read it. Say which rule set you're applying.

### Step 2: Check each changed item

The branch diff names what changed but not the values. For each modified item, `get` it on the branch (`branchId`) **and** on main, and compare to see exactly what this branch adds or removes. New items only need the branch read. Read properties with `get` (`type: "property"`) for their allowed values and per-event restrictions; the event read doesn't include them.

**Review what the branch introduces.** A problem that already exists on main (an event name that predates the current naming rule, say) is not a blocker for this branch. List it under "Pre-existing" so it isn't lost, without holding up the review.

Check:

| Check | How | Severity |
|---|---|---|
| Audit rules (naming format, casing, allowed words, and the other configured checks) | The branch `audit` from Step 1; compare against `workspace_config` by hand only when the audit gives counts alone | Blocker when the audit says it blocks the branch, otherwise Suggestion |
| Description present and readable | Says when the event fires in plain language, no engineering jargon (data-designer: Descriptions) | Blocker if the description rule is Enforce, otherwise Suggestion |
| Matches neighbouring items | Filter `search` on the same category or event; compare structure, tense, prefix style | Suggestion |
| Duplicate of something on main | Semantic `search` (and exact `get`) on main for the same meaning | Blocker if it's a clear duplicate, Question if it might be |
| Property types and constraints | No `any` where enforced; finite values have allowed values; no Booleans for state that may grow | By rule, otherwise Suggestion |
| Near-duplicate allowed values | A new value that differs from an existing one only by case, spacing, or spelling (`"Product Details"` next to `"Product details"`); also check what other events send on the same screen | Blocker: two spellings split the data |
| Where it fires | New events have a trigger (journey or standalone) with a description; conditions only use allowed values | Question if missing |
| PII | Properties that look like personal data have a PII declaration if the workspace requires it | By rule |
| Ownership and categories | Required owners and categories present | By rule (usually Warning) |
| Destructive changes | Removed events or properties, renamed items without a name mapping, changed types | Question: confirm downstream impact |

**Gateway workspaces.** If `search` with `itemType: "gateway"` returns any gateways, also check where each changed event ends up. The branch diff doesn't show routing: changes to gateway inputs, outputs, allow-lists, renames, and source-to-destination connections don't appear in `all_changes`. So:

1. For each changed event, `get` it with `checkpoint: {kind: "destination", destinationId}` for the destinations it routes to, **on the branch and on main**, and compare.
2. If the branch description or name suggests routing work ("send X to Snowflake"), also `get` the gateways involved (`type: "gateway"`) on the branch and on main and compare their outputs.

| Check | How | Severity |
|---|---|---|
| A property the branch adds never reaches the destination that needs it | Checkpoint read lists it under "Dropped by the allow-list" | Blocker when the branch's stated purpose depends on it, otherwise Question |
| An event newly reaches a destination | Reaches on the branch, not on main | Question: confirm it's intended |
| Personal data newly reaches a destination | Checkpoint PII list grows | Blocker if not declared or not intended |
| Arrives under the wrong name | "Arrives as" doesn't match the destination's expected name (e.g. Meta's standard events) | Suggestion |

Only report findings with evidence: the rule, the neighbouring item, or the duplicate's name. "Could be clearer" without a better alternative is not a finding.

### Step 3: Report

```
Review: <branch name> — <link>
Verdict: 2 blockers, 3 suggestions, 1 question

BLOCKERS
1. Event "checkout_done" — name breaks the workspace format <OBJECT>_<ACTION> (Enforce).
   Fix: rename to "CHECKOUT_COMPLETED" (matches CHECKOUT_STARTED in the same category).
2. Property "price" — type "any" is blocked by the workspace rules.
   Fix: "float"; the values in the triggers are all numeric.

SUGGESTIONS
...

QUESTIONS
1. "Order Placed" (new) looks like the existing "ORDER_COMPLETED" on main. Same action?

PRE-EXISTING (on main, not introduced by this branch)
1. "CHECKOUT_STARTED" — STARTED isn't in the workspace's allowed actions.

Summary for reviewers
<3–6 plain-language lines: what this branch adds, why, where it fires>
```

### Step 4: Hand the findings back

The MCP can't post comments on a branch yet. Give the findings grouped by item so the user can paste them into the branch's comments in Avo, or into their own review tool. Offer two follow-ups:

- Fix the findings: the user approves which ones, then the data-designer skill's small-edits flow (or bulk-edit-tracking-plan for many items) applies them on the same branch.
- Hand off to developers: the branch-implementation-handoff skill.

---

## Gotchas

- **Reviewing your own branch and passing it.** If the agent wrote the branch earlier in the conversation, it is anchored on its own choices. Re-read every item against the rules as if a stranger wrote it.
- **Severity inflation.** Only Enforce rules and clear duplicates are blockers. Taste is a suggestion.
- **Claiming who approved.** The MCP returns an approval count and a reviewer list, not which reviewer approved.
- **Approving or merging.** Never. Review is advice; the decision stays with the humans in Avo.
- **Raw rich-text descriptions.** Some descriptions come back as `{"ops":[{"insert":"…"}]}`. Review the `insert` text, and don't report the JSON wrapper as a description problem.
