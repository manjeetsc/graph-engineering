---
name: graph-migrate
description: Applies the same kind of change across many files or items — one node per item, each making its change and getting checked before it's counted as done, with a consolidated report of what passed, what stopped blocked or stagnant, and what never ran. Use when asked to apply a change across many files, rename or replace something repo-wide, or run the same transformation over a list of items.
---

# Graph Migrate

A ready-made `graph-run` configuration where the node count isn't fixed ahead of time like
`graph-review`'s dimensions — it's however many items are actually in the list, decided when
the graph is planned, not written into the skill itself.

## Arguments

| You say | What it does |
|---|---|
| the change, plus what defines the item list (a glob, a list of files, a query) | applies it across every matching item |
| `--dry-run` | lists the items it would touch and stops |
| `--max-parallel N` | cap on how many item nodes run at once (default: a sane number for the list size, not unlimited) |
| `--on-fail skip\|stop` | if an item never passes its check after retries, keep going on the rest (default) or stop the whole run |

## Before running

If the list has one or two items, just make the change directly — the per-item node,
check, and repair machinery earns its cost at real scale, not on something you'd do faster by
hand.

## Step 1 — enumerate the list

Before anything runs, produce the actual list of items this will touch and report the count.
Nothing gets included or excluded silently — if the list looks wrong, this is the point to
catch it, before any node has started.

## Step 2 — one node per item

Every item gets its own node, all sharing the same contract (what the change should look like
once correctly applied) and the same instructions — but each node is scoped only to its own
item. No node reads or touches another item's node.

## Step 3 — check each item

Every node's result gets checked against the shared contract: did the change actually apply
the way it was supposed to, and is the item still otherwise intact.

## Step 4 — repair, per item

An item that fails its check gets retried — just that item's node, with the specific failure
fed back in. Nothing about the other items is touched by one item's retry. Same judgment call
as `graph-run`'s Step 4: keep retrying while each attempt is actually closing the gap; the
moment a retry fails for the same underlying reason the last one did, that's not converging —
it escalates to `graph-plan`, once, same as any other node, not tried a third time unchanged.
If the replanned version still doesn't resolve it, that item is blocked — needing a human look,
nothing left to try that hasn't already been tried. An item where several rounds instead recur
with no real change at all — not new failures, not progress, the same state just repeating —
is stagnant rather than blocked, but gets the same treatment either way: stop retrying it,
report it plainly, move on to the rest of the list.

If the list itself is too large to finish in one sitting, this is exactly the kind of task
`graph-plan`'s "Optional: remembering across separate runs" was written for — a progress
ledger this skill's own executing side writes to, one entry per item, lets a second invocation
pick up the remaining items cold instead of starting the list over.

## Step 5 — assemble

One consolidated report, item by item: which passed, which stopped blocked or stagnant instead
(named as one or the other, not folded into one vague "didn't work" bucket), and which were
never attempted at all because `--on-fail stop` ended the run early.

## Examples

```
Rename this function everywhere it's used, --dry-run first.
Apply this same config change across every service directory.
Update the import style in all files matching src/**/*.ts, --max-parallel 8.
```

## What this refuses to do

- Won't force through a change on an item that failed its check after retries.
- Won't touch anything outside the list produced in Step 1.
- Won't ignore `--max-parallel` and fan out unlimited nodes regardless of list size.
