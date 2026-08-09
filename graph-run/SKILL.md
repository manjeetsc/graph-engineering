---
name: graph-run
description: Actually executes a task as a graph. Runs each node in parallel where nothing depends on it yet and in sequence where it does, checks every node's output against its contract, repairs and retries only the node that failed (never restarts the whole thing), and assembles the final result. Use when asked to run or execute a plan from graph-plan, or to just get a multi-step task done without wiring the graph up by hand.
---

# Graph Run

Takes a plan and actually does the work. Nothing here is a description of what should
happen; every step in this skill produces a real result.

## Two ways in

- **You already have a plan**: freshly produced by `graph-plan`, loaded from a file
  `graph-plan` saved earlier for reuse, or written by hand in the same shape: context, nodes
  with contracts, checks, and dependencies, an assembly rule, and ownership settled for
  anything nodes might collide on. Execute it as-is, whichever source it came from.
- **You just have a task**, and it's small or well-understood enough not to need a separate
  planning pass. Run `graph-plan`'s Step 0 through Step 6 silently first, then continue below
  with what it produces. If Step 0 concludes this doesn't need a graph, stop there and just
  do the task directly. Don't force an execution that wasn't warranted.

## Step 1: set up context, scoped per node

Gather whatever the plan's context calls for once, up front, so it's not re-derived from
scratch by every node. But hand each node only the slice its own contract actually needs, not
the whole pool by default: a security node doesn't need the style guide, a writer doesn't
need the security findings. If two nodes' scoped context overlaps, that's fine; if a node's
scope is "everything," that's usually a sign its job in the plan is still too broad.

## While it runs: say so

Don't go quiet until the final report. Before Step 2 starts, show the plan as a short
checklist (one line per node, in run order, so it's clear what's about to happen). As
execution proceeds, update it: mark a node running when it starts, passed or needs-repair
when it finishes its check. This applies in every mode, including auto: auto only changes
whether *planning* pauses to ask something, never whether *running* stays visible. Silence
during execution isn't a neutral default, it's a missing status update.

```
[✓] researcher   - passed
[✓] writer       - passed
[↻] reviewer     - didn't pass, repairing: "missing X"
[ ] assembler    - waiting on reviewer
```

If a node goes back for repair, say that plainly when it happens, not just in the final
summary: which node, why, and which attempt this is. A loop-back is exactly the kind of
thing that shouldn't happen silently.

This can run as a background job (you don't have to sit and watch a spinner), but "runs in
the background" means you get the final result without babysitting it, not that progress goes
unreported along the way. Post the checklist update as each node actually finishes, not
retroactively once everything is done.

## Step 2: run the nodes

Start every node that has nothing left to wait on; several of these can run at once. As each
one finishes, start whatever was waiting only on it. A node with no dependents doesn't block
anything; a node several others depend on becomes a real join point where their outputs come
together before the next stage starts. This is dependency order, not a fixed sequence: the
plan's dependency list decides it, not the order the nodes happen to be written in.

"Nothing left to wait on" means nothing in the dependency list *and* no unresolved ownership
collision: two nodes about to touch the same file or resource at once are not actually clear
to run together, whatever the dependency list says. If the plan didn't settle that, settle it
here before dispatching both: sequence them, or don't start the second until the first
releases what it's holding.

## Step 3: check each result

Every node's output gets checked the way its contract said it should: a second pass trying to
disprove it, or a deterministic check for what the contract required. A result that passes is
kept and handed to whatever depends on it. A result that fails goes to repair before anything
downstream of it is allowed to start.

Zoomed in, this is the same move a single well-built agent already makes on its own: do the
work, get checked by something independent, don't let the same pass praise itself. A graph
doesn't invent a new kind of check; it just runs that move once per node instead of once for
the whole task.

## Step 4: repair, don't restart

A failed node gets a real retry, with the specific reason it failed fed back into it, still
just that one node. Nothing that already passed gets touched, and nothing downstream starts
until the repaired node passes its check too. Narrate this the moment it happens, per "While
it runs" above; a retry that only shows up in the final report, after the fact, is too late
to be useful.

Before retrying, tell two different failures apart, because they need different fixes:

- **The node did its job badly**: an execution problem. Retry it: same job, the specific
  failure fed back in.
- **The node was given the wrong job**: a planning problem wearing an execution problem's
  clothes. `graph-run` doesn't replan. Hand this back to `graph-plan` (or, inside `graph`,
  back to its planning step) to fix the actual contract or node breakdown, then resume
  execution from there. Say plainly when this happens: this isn't another retry, it's going
  back further than that.

**How to decide which, and when to stop (a judgment call, not a fixed count):** keep
repairing as long as each attempt is actually converging on the problem: the specific reason
it failed is getting addressed, not just tried again with different wording. That's still an
execution problem; keep going. The moment a repair attempt fails for the *same underlying
reason* it just failed for, it isn't converging; that's the signal it was a planning problem
the whole time, and the fix is to escalate to `graph-plan`, not to try a third worded-slightly-
differently attempt at the same level. If escalating to planning and trying again *still*
doesn't resolve it, that's the actual stop: report exactly what's blocking it, as unresolved,
rather than escalating again with nowhere left to go. In practice that's usually one retry, or
one retry plus one escalation, not because a number was picked in advance, but because
that's how many distinct things are left to try before repeating something that already
didn't work.

**The backstop, separate from that judgment call:** none of this is allowed to run forever
even in a pathological case, so carry a hard ceiling underneath the judgment-based rule, on
total attempts or total spend for one node's repair chain, generous enough that it's never
what actually ends a normal run. The judgment call above is what decides when to stop in
practice; the ceiling exists purely so a wrong assumption can't spin unbounded if it does.
Hitting the ceiling looks the same as the judgment call concluding "not resolvable here": stop
and report unresolved, don't keep going, and don't ship a failure as if it had passed.

## Step 5: assemble, then check the whole

Once every node has either passed or come back unresolved by the rule above, combine the
passing outputs per the plan's assembly rule into the final deliverable, then run the
whole-result check from `graph-plan`'s Step 6 against that combined result. Passing every
node's own contract doesn't guarantee the assembled whole is right; this is what catches the
failure no single node could see on its own (code that builds per-file but not together, a
brief that's individually sourced but misrepresents the research once combined).

This check loops too, under the exact same rule as Step 4: if it fails, trace the failure to
the specific node(s) responsible, repair them (same execution-vs-planning judgment call, same
backstop underneath it), reassemble, and check the whole again. Keep going while it's
converging, stop and report when it isn't. A check that only ever runs once isn't
verification, it's a formality.

## Report

The closing summary, after the checklist has already shown the run happening live: which
nodes ran, which passed on the first try, which needed repair (and what the repair actually
fixed, and whether that took a plain retry or a planning escalation), which came back
unresolved, and whether the whole-result check from Step 5 passed, including how many repair
cycles that took, or why it stopped without passing. None of that gets silently dropped from
the final result. Nothing beyond what the plan asked for gets changed; if the task was
"review this" rather than "fix this," nothing is auto-applied.

## Cost

Not a fixed formula: see Step 4's judgment rule and backstop. Cost tracks how much genuine
work the problem needed, bounded underneath by a ceiling that should essentially never be what
actually ends a normal run. A graph built this way rarely needs a separate spending limit
bolted on top; the stopping rule already is one.

## Refuses to

- Restart a node that already passed just because a sibling somewhere else failed.
- Let a node check its own output: the check is always a different pass.
- Silently drop a node that never passed. It's reported as unresolved, not omitted.
- Go quiet between the plan and the final report. Progress is shown as it happens, not
  reconstructed afterward.
- Keep retrying once an attempt stops converging, on any loop, node-level or whole-check.
  That's the stop signal, whether or not the backstop is anywhere near being reached.
- Treat a planning-shaped failure as another execution retry. Failing for the same underlying
  reason means the contract goes back to planning, not another attempt at the same wording.

For a plan produced ahead of time, see `graph-plan`. For ready-made graphs that skip the
planning step entirely, see `graph-review`, `graph-research`, `graph-content`,
`graph-migrate`, and `graph-compare`; each is this same mechanism, pre-configured for one
common job.
