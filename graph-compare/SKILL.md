---
name: graph-compare
description: Tries two or more different approaches to the same problem in parallel, each as a full independent attempt, then has a separate judge node score them against stated criteria and recommend one, or a synthesis of the strongest parts of more than one. Use when asked to compare approaches, try something multiple ways, or decide between options by actually attempting each one rather than reasoning about them abstractly.
---

# Graph Compare

A ready-made `graph-run` configuration where the nodes are full independent attempts, not
just findings or research notes, and the fan-in at the end is a judge, not a synthesis writer
or a fact-checker.

## Arguments

| You say | What it does |
|---|---|
| the problem | tries it a few different ways and recommends one |
| `--approaches x,y,z` | use these specific approaches instead of letting it propose some |
| `--criteria "..."` | state what "better" means explicitly instead of leaving it implicit |

## Before running

If there's genuinely one obvious right approach, just do that one. This earns its cost when
the outcome is actually uncertain, not when comparing is just a formality.

## Step 1: name the approaches

Two to four distinct approaches to the same problem, each substantial enough to be a real,
independent attempt rather than a variation on the same idea. `--approaches` overrides this
outright.

## Step 2: state the criteria before anything is attempted

Same reasoning as `graph-content`'s bar: write down what "better" means before any attempt
exists, so the judge in Step 4 is checking attempts against a standard set in advance, not
inventing one afterward that happens to favor whichever attempt reads better on the surface.

## Step 3: run every attempt in parallel, independently

Each approach is its own node, and none of them sees what the others produced while it's
building its own, comparing what actually got built, not one attempt anchored on another.

## Step 4: check each attempt

Before the judge sees anything, every attempt gets checked against the same rule any node
does: if it produces or changes code, the check means actually building and running it, not
reading it and guessing whether it works. An attempt that fails its own check goes through
standard repair: retried with the specific failure fed back, escalated to `graph-plan` if the
same failure recurs, reported blocked or stagnant rather than silently dropped if neither
resolves it. Only attempts that actually work reach the judge; comparing a broken attempt
against a working one isn't a real comparison, it's a foregone conclusion dressed up as one.

## Step 5: judge

A separate node, never one of the attempt nodes, starting clean with no attempt's own
reasoning carried over, scores each checked attempt against Step 2's criteria and either
recommends one outright or proposes a synthesis that grafts the strongest parts of more than
one attempt together, explaining the reasoning either way. If every attempt failed its Step 4
check, say so plainly instead of forcing a recommendation among broken options.

## Report

Every attempt, not just the winner (you can see what was actually tried, including which
ones failed their own check and how), plus the judge's scoring against each stated criterion
and the final recommendation with its reasoning.

## Examples

```
Compare two ways to implement this, tell me which is better.
Try three approaches to this migration and recommend one, --criteria "least
  downtime, easiest to roll back."
Compare --approaches "rewrite from scratch,incremental refactor" for this module.
```

## What this refuses to do

- Won't let an attempt's own node also act as part of the judge, or let the judge start with
  any attempt's own reasoning attached.
- Won't send an unchecked attempt to the judge: code attempts get built and run first.
- Won't pick a winner without criteria stated first.
- Won't drop the losing attempts, or the ones that failed their own check, from the report:
  they're shown, just not chosen.
