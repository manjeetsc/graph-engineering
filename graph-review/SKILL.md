---
name: graph-review
description: Runs a graph-structured review of the current changes (or a given diff/PR): several specialized reviewer nodes working in parallel, each finding independently double-checked by a skeptical second pass, reported back with evidence. Never edits anything. Use when the user asks to review a PR/diff/branch thoroughly, wants a multi-angle code review, or wants to see graph-run in action on something real.
---

# Graph Review

A ready-made `graph-run` configuration: point it at a diff, and it runs several specialized
reviewer nodes side by side, then has each of their findings independently double-checked
before anything gets reported to you. Nothing here ever changes your code; it only tells you
what it found.

Distinct specialties (each reviewer node owns one concern) plus an otherwise-overloaded
verifier (one reviewer can't credibly judge correctness, security, and style all at once) are
exactly the two signals from `graph-plan`'s Step 0 that justify a graph in the first place.
This skill is what answering "yes" to those looks like, already wired up.

## Arguments

| You say | What it reviews |
|---|---|
| (nothing) | your current uncommitted changes |
| a commit range, e.g. `main..HEAD` | that range |
| a PR number | that PR's diff |
| `--dimensions x,y,z` | only the concerns you name, instead of the defaults |
| `--depth thorough` | adds a ranking pass and a "what got missed" pass on top of the default single pass |
| `--dry-run` | shows the planned reviewer nodes without actually running anything |

## Before running

Confirm there's an actual diff to look at. If it's trivial (a couple of lines, a config
tweak), say so and suggest a normal read-through instead of spinning up a graph for it. A
graph earns its cost; a two-line change doesn't need one.

## Step 1: pick the reviewer nodes

Default set: correctness, security, simplification, test coverage. Add or swap based on what
changed (touching auth code pulls in a security-focused node even if not asked; a UI-only
diff might drop test coverage). `--dimensions` overrides this outright. Cap it around five or
six unless `--depth thorough` was asked for. More nodes than that stops adding signal and
starts adding noise and cost.

## Step 2: run it

Each reviewer node's contract: findings in a consistent shape (file, line, what's wrong, why
it matters), not a paragraph. Every finding then gets checked by a second, independent node
whose only job is to try to disprove it. A finding that survives a skeptical second look is
far more likely to be real than one that was only ever looked at once.

On `--depth thorough`, add a pass that ranks the surviving findings by severity, and a last
pass that asks what wasn't covered (a file that got skipped, a concern nobody checked), so
nothing is silently left out of the report.

This runs as a background job. You'll get a result rather than watching it live, and you can
check in on it while it's running. Total cost is bounded by construction: node count (capped
around five or six) times pass count (one by default, up to three on `--depth thorough`),
small enough that a review this size doesn't need a separate spending cap the way a
longer-running, open-ended graph would.

## Step 3: report

At default depth, findings come back in the order they were found, each with the evidence
behind it (not just an opinion, but what was actually found and why it matters), and the
report states which dimensions and files were actually checked, so the scope is never left
implicit. On `--depth thorough`, findings are additionally ranked by severity, and a dedicated
pass reports anything a dimension or file didn't get covered. Nothing is auto-fixed; you
decide what to act on.

## Examples

```
Review my current changes.                        → uncommitted diff, default dimensions
Review PR 42 for security and correctness only.    → --dimensions security,correctness
Do a thorough review of main..HEAD.                → that range, --depth thorough
Show me what a review of this would look like       → --dry-run: lists the planned
  before you actually run it.                         reviewer nodes, runs nothing
```

## What this refuses to do

- Won't grade a finding with the same reviewer node that raised it: the check is always a
  different pass.
- Won't run on a trivial diff just because it was asked to. It'll say so and suggest a plain
  read instead.
- Won't silently skip a file or concern without saying so in the final report.
