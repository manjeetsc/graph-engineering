---
name: graph-build
description: Builds something in a strict, ordered sequence of specialist stages (scaffold, then implement, then test, then document), each stage depending on exactly the one before it, no fan-out, no designed loop-back edge. Use when a task is genuinely a fixed pipeline of steps that must happen in order, rather than something that benefits from parallelism or a revision cycle.
---

# Graph Build

A ready-made `graph-run` configuration for the plainest graph shape there is: a straight
chain. Every other ready-made skill here has a distinctive shape: `graph-review`'s parallel
nodes, `graph-research`'s fan-out into one synthesis, `graph-content`'s designed loop-back,
`graph-migrate`'s one-node-per-item, `graph-compare`'s parallel attempts feeding a judge. This
one has none of that, on purpose: node B depends on exactly node A, node C depends on exactly
node B, nothing branches and nothing was designed to route backward. It's also, in practice,
the single most common shape a real task actually has.

## Arguments

| You say | What it does |
|---|---|
| what to build | runs the default stage chain against it |
| `--stages x,y,z` | use this stage list instead of the default one |

## Before running

If the steps don't have a genuine strict dependency order (two of them could really happen
in parallel, or in either order), this isn't the right shape; a graph with an invented
dependency to make it look sequential is worse than one that's honest about being parallel.
And if there's only one or two steps, just do them directly.

## Step 1: the stage chain

Default stages: scaffold (set up the structure: files, module layout, whatever the thing
needs to exist before it has content) → implement (write the actual logic) → test (write and
run a real test suite against it) → document (write what a future reader needs to understand
it). Each stage is a node whose only dependency is the stage immediately before it: a chain,
not a tree, not a DAG with branches. `--stages` overrides the default list outright, but the
same rule applies to whatever list you use: each stage depends on exactly the one before it.

## Step 2: a contract per stage, checked before the next one starts

Same rule as anywhere else in this set: a contract concrete enough to check without an
opinion. The "test" stage's contract in particular means actually running the tests, real
execution, not a read-through, since a pipeline that produces code and never runs the tests
it wrote is just theater with extra steps. A stage only hands off to the next once it's passed
its own check; nothing downstream starts against an unverified upstream stage.

## Step 3: repair, the standard way, not a bespoke loop-back

If a stage fails its check, it gets `graph-run`'s ordinary per-node treatment: retried with
the specific failure fed back, escalated to `graph-plan` if the same failure recurs, stopped
(passed / blocked / stagnant) by the same three-way stop as anywhere else. This is exactly
what makes the shape different from `graph-content`: there's no edge in this plan that was
*designed* to send work backward to an earlier stage as expected, ordinary routing. If "test"
turns up a bug that's really in "implement," that's an escalation: `graph-plan` reconsidering
an earlier stage's contract, not a loop-back this skill built in as normal traffic.

## Step 4: assemble

Once every stage has passed, the final deliverable is what the chain produced end to end,
usually just the sum of what each stage wrote, since a strict chain doesn't need a separate
synthesis step the way a fan-in does. The whole-result check from `graph-plan`'s Step 6 still
applies: build and run the assembled result once more, since stages that each passed alone can
still fail to hold together as a whole.

## Report

Which stage produced what, which needed repair and how, and, if a stage stopped blocked or
stagnant instead of passing, exactly where the chain actually broke, since in a strict
sequence that's also where everything after it stopped.

## Examples

```
Build a new export-to-CSV feature: scaffold, implement, test, document.
Add this endpoint end to end, --stages "implement,test" (skip scaffold, it already exists).
```

## What this refuses to do

- Won't run two stages concurrently to save time when their contracts genuinely depend on
  each other. That's not this shape.
- Won't let a stage's check be a read-through when the stage produced code. It gets run.
- Won't quietly patch an earlier stage from inside a later one. A bug found downstream that
  actually belongs upstream is an escalation, handled by `graph-plan`, not an ad hoc fix.
- Won't report a broken chain as finished. The report names exactly which stage stopped it.
