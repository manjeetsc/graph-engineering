---
name: graph-research
description: Researches a question by fanning out across several angles or sources in parallel, synthesizing the results into one written brief, then independently fact-checking the brief's own claims against what was actually found. Use when asked to research a topic, compare options, gather information from multiple angles, or produce a sourced summary or brief.
---

# Graph Research

A ready-made `graph-run` configuration for research. The graph shape here is different from
`graph-review`'s: instead of several independent nodes reporting side by side, the research
nodes feed into one synthesis node — a real join point, not just parallel work reported
separately — and the synthesis itself gets checked, not just the raw research.

## Arguments

| You say | What it does |
|---|---|
| a question or topic | researches it end to end |
| `--angles x,y,z` | use these specific angles instead of letting it pick them |
| `--depth thorough` | on any claim the fact-check can't confirm, run a follow-up research pass instead of just flagging it |

## Before running

If the question is narrow enough that one direct search or lookup answers it, say so and just
answer it — a graph earns its cost through genuine breadth; a single-fact question doesn't
need one.

## Step 1 — pick the angles

Break the question into sub-questions distinct enough to research independently and in
parallel — for a factual comparison that might be "what/how/tradeoffs/examples," for
something else entirely different angles will fit. Each angle is a node with one job.
`--angles` overrides this outright.

## Step 2 — research each angle

Each angle node's contract: a set of claims, each one tied to where it came from. Not a
paragraph of prose — a claim, and its source, repeated for everything the node found. This is
a deterministic check, not a judgment call: does the output actually contain claim-and-source
pairs, or did the node hand back unsourced prose. An angle node that fails its own contract
gets the same repair any node does — retried with the gap fed back, escalated to `graph-plan`
if the same problem recurs, reported blocked or stagnant if neither resolves it. These nodes
don't depend on each other, so they run in parallel.

## Step 3 — synthesize

One node depends on every angle node *passing* its check, not just finishing — a real fan-in,
not just several reports sitting side by side. Its contract: one coherent brief that actually
uses the claims from every angle that made it through, not a copy-paste of each angle's notes
back to back. An angle that came back blocked or stagnant doesn't block synthesis — it's left
out of the brief and named explicitly in the report instead.

## Step 4 — fact-check the synthesis

A fresh node — starting clean, none of the synthesis node's own reasoning carried over —
checks the brief's claims against what the angle nodes actually reported, not against the
world, against the research that was already done. This catches a different failure than Step
2's sourcing: a claim that drifted or got overstated during synthesis, even though its source
was solid. This check always runs, at every depth — skipping it is exactly the failure mode
this graph exists to prevent, and it's cheap enough not to gate behind a flag.

On `--depth thorough`, anything the fact-check can't confirm goes back to the relevant angle
node for a follow-up pass instead of just being flagged — a designed loop-back edge, not an
escalation. That follow-up gets the same three-way stop as anything else: confirmed (done),
blocked (the angle genuinely can't confirm it, escalate to `graph-plan` if it's worth
reconsidering whether the claim belongs at all), or stagnant (repeated follow-ups turn up
nothing new) — either of the last two ends in the claim being flagged, not chased forever.

## Report

The brief, which angles were actually covered (and which came back blocked or stagnant and
were left out), and any claim the fact-check couldn't confirm — listed explicitly, not
silently dropped or quietly removed from the brief.

## Examples

```
Research the tradeoffs between X and Y.
Compare three approaches to Z and tell me which fits our case.
Research this thoroughly — --depth thorough, so unconfirmed claims get a follow-up
  pass instead of just being flagged.
```

## What this refuses to do

- Won't present a claim with no traceable source behind it.
- Won't skip the fact-check pass, at any depth, or let it start with the synthesis node's own
  reasoning attached — it starts clean.
- Won't silently drop an angle that came back blocked or stagnant — it's named in the report
  instead.
- Won't chase an unconfirmable claim through endless follow-up passes — the same three-way
  stop applies here as anywhere else.
