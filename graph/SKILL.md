---
name: graph
description: The single entry point for "just do this as a graph": plans a task (graph-plan) and then executes it (graph-run) in one go, in one of two modes. Guided mode (the default) asks about specific points during planning only when something is genuinely unclear, then runs straight through with no further pauses. Auto mode resolves every planning decision itself and runs straight through, reporting what it decided afterward. Use when asked to run a task end to end without invoking graph-plan and graph-run separately, or when asked for "auto mode" or "guided mode."
---

# Graph

One command, plan through finished result. This is `graph-plan` and `graph-run` wired
together. Use it when you don't want to invoke the two separately, and pick a mode based on
how much you want to see before it runs.

## Modes

Both modes run execution the same way: once planning is settled, nothing pauses again until
the result is back. The difference is entirely in what happens during planning, before any
node has run.

| Mode | During planning | Best for |
|---|---|---|
| `--guided` (default) | Only asks when something is genuinely unclear: which angle matters most, what a contract should require, whether a node should exist at all. If nothing about the plan is actually ambiguous, it doesn't ask anything and goes straight into execution, same as auto. | The general case: costs nothing when the plan is obvious, catches real ambiguity when it isn't |
| `--auto` | Resolves every one of those same points itself, using its own best judgment, and never asks | Something well-understood, lower-stakes, or where you've explicitly asked for zero friction even on genuinely ambiguous points |

Guided is the default because it's never worse than auto: when the plan is unambiguous, the
two behave identically. It only diverges, by asking, at the exact points where a wrong
assumption would otherwise run all the way through execution before anyone noticed.

## Step 1: check whether this needs a graph at all

Same as `graph-plan`'s Step 0, in both modes, always. A mode choice doesn't change whether a
graph is warranted. It only changes how the planning that follows behaves.

## Step 2: plan

Run through `graph-plan`'s steps: context, nodes, contracts, checks, assembly rule.

- **Guided**: if building the plan hit a point that was genuinely uncertain (not just any
  decision, a point where more than one reasonable answer existed), stop and ask about that
  specific point ("should the security angle be its own node or folded into correctness?", "is
  a 400-word cap the right bar for this draft?"), not "does this look okay?" Adjust the plan
  based on the answer and keep building. If nothing in the whole plan was actually uncertain,
  nothing gets asked, and this proceeds straight to Step 3 exactly like auto mode would.
- **Auto**: resolve those same uncertain points itself, using its own best judgment, and never
  pause. Don't skip past them silently. Note each judgment call made, so it shows up in the
  final report even though nobody was asked in the moment.

Once the plan leaves this step (confirmed, self-resolved, or simply unambiguous to begin
with), nothing pauses again. Step 3 always runs straight through in both modes.

Before moving on, say what the plan actually is (as a short checklist, one line per node)
even when nothing was asked, even in auto mode. "Nothing to clarify, here's what's about to
run: ..." is the floor. Going straight from silence into execution is not acceptable in either
mode; the only difference between modes is whether getting to that checklist involved a
question.

## Step 3: run

Once the plan is settled (confirmed in guided mode, self-resolved in auto mode), execute it
exactly as `graph-run` does, all of it: nodes in dependency order, each checked against its
contract, failures repaired and retried (and announced when it happens, not just summarized
after) without restarting what already passed; a node that keeps failing for the same reason
escalated back to planning rather than retried a third time with different wording; the
assembled whole checked too, not just its pieces, looping the same way until it passes or
genuinely stops converging, not against a fixed number of tries. Auto mode changes nothing
here: "auto" means planning didn't stop to ask, not that execution goes dark or that any of
this gets skipped.

## Report

Everything `graph-run` reports (what ran, what passed, what needed repair, what never passed),
plus: in guided mode, what was confirmed or changed during planning; in auto mode, the list of
judgment calls made without asking, so nothing that shaped the outcome is invisible just
because nobody was in the loop for it.

## Examples

```
Run this as a graph.                          → guided (default): plans, asks about
                                                   anything genuinely uncertain, then runs
Just do it, auto mode.                        → --auto: plans and runs straight through,
                                                   reports what it decided along the way
Plan and run: research X, then write a brief   → guided, since it's the default; add
  and have someone else check it.                --auto if you don't want the pause
```

## Refuses to

- Skip the "does this need a graph" check in either mode. A mode only changes how planning
  behaves once a graph is actually warranted.
- Ask an unanswerable or vague question in guided mode ("does this look good?"). Every pause
  is for a specific, decidable point.
- Make an auto-mode decision that's hard to undo (something destructive, something that
  contacts someone outside this conversation, something that spends real money) without
  flagging it clearly, even though auto mode doesn't pause for ordinary judgment calls.

For the two stages on their own, see `graph-plan` and `graph-run`. For ready-made graphs that
skip planning language entirely, see `graph-review`, `graph-research`, `graph-content`,
`graph-migrate`, and `graph-compare`.
