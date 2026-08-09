---
name: graph-content
description: Drafts a piece of writing, then hands it to an independent reviewer node with a real bar to clear. If it doesn't clear the bar, it goes back to the writer with specific feedback and gets revised, looping until it passes or revisions stop making progress. Use when asked to write something that should be checked before it's final (a document, a post, an explanation), not just produced in one pass.
---

# Graph Content

A ready-made `graph-run` configuration for writing that needs a real check, not just a
first draft. The graph shape here is different again from `graph-review` (parallel nodes) and
`graph-research` (fan-out into one synthesis): this one is mostly sequential, with a loop-back
edge: a node that can send work backward to an earlier node, not just forward.

## Arguments

| You say | What it does |
|---|---|
| what to write, and for whom | drafts, reviews, revises until it clears the bar |
| `--bar "..."` | state the acceptance bar explicitly instead of inferring one |
| `--max-revisions N` | force a hard revision ceiling, if you specifically want one tighter than the default judgment call below |

## Before running

If this is short enough that you'd read it yourself in the time it takes to describe what
"good" means, just write it directly. The loop-back machinery earns its cost on something
long or consequential enough that a bad first draft would actually cost you something to
catch late.

## Step 1: state the bar

Before anything is written, write down what "passes" actually means for this piece: the
audience, the length, what it must cover, what would make it wrong. This becomes the
reviewer node's contract. A vague bar here is the single most common reason this loop never
converges. It isn't that revisions don't happen, it's that "better" was never defined enough
to check against.

## Step 2: draft

The writer node produces a full draft against the bar from Step 1 and whatever context was
given about the topic.

## Step 3: review against the bar, not against taste

The reviewer node is a different pass than the one that wrote it, checking specifically
against Step 1's bar, not offering a general opinion. Output is a clear pass, or a specific
list of what's missing or wrong, each item concrete enough that the writer node could act on
it without guessing what was meant.

## Step 4: the loop-back edge

If Step 3 doesn't pass, its findings go back to the writer node as the next input, and Step 2
through Step 3 run again. That's a backward edge: routing that sends work back to a step
that already ran, instead of ahead to one that hasn't. Say so when it happens ("didn't pass,
sending it back with: [findings]"). The same live-checklist rule `graph-run` follows for any
repair applies doubly here, since a loop-back is the one kind of routing in this whole set of
skills most likely to look like nothing happened if it isn't narrated.

Stopping is the same judgment call as `graph-run`'s Step 4, not a fixed count: keep revising
as long as each round is actually closing the gap the reviewer named. If a revision comes back
with the same finding the reviewer already raised (not a new one, the same one, still
unresolved), that's not converging. Before just stopping, though, ask the same
execution-vs-planning question `graph-run` asks: is the writer failing to hit a clear bar
(an execution problem, genuinely stop and report), or is the bar itself unmeetable as
written, too vague, contradictory, or asking for two things that trade off against each
other (a planning problem, back in Step 1)? A bar that's the actual source of a stuck loop
should get revised once, same as any other contract would, rather than exhausting revisions
against a target that was never fair to begin with. `--max-revisions` is there if you want a
hard ceiling tighter than that judgment call, but the default isn't a number picked in
advance, it's however many rounds (and at most one bar revision) are each still making real
progress. Either way it stops, and stopping without a pass is reported as-is, with the
reviewer's last set of findings attached, rather than shipped as if it passed.

## Report

The final draft, whether it passed on its own or after revision, how many revision rounds it
took, and, if it stopped without passing, why: the same finding recurring unresolved, or a
`--max-revisions` ceiling, either way with the reviewer's outstanding findings attached so you
can see exactly what wasn't resolved.

## Examples

```
Write a short explainer of X for someone who's never heard of it.
Draft a project update for the team, --bar "covers what shipped, what's blocked,
  and next week's plan, under 200 words."
Write this, but don't let it go back and forth more than twice. → --max-revisions 2
```

## What this refuses to do

- Won't let the writer node review its own draft: the check is always the separate reviewer
  node.
- Won't ship a draft that stopped without passing, without saying why.
- Won't send it back a second time for the same unresolved finding without treating that as
  the stop signal it is.
- Won't review against an unstated bar: Step 1 always happens first, even implicitly.
