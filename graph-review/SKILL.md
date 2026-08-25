---
name: graph-review
description: Runs a graph-structured review of the current changes (or a given diff/PR) — several specialized reviewer nodes working in parallel, each finding independently double-checked by a skeptical second pass, reported back with evidence. Never edits anything. Use when the user asks to review a PR/diff/branch thoroughly, wants a multi-angle code review, or wants to see graph-run in action on something real.
---

# Graph Review

A ready-made `graph-run` configuration: point it at a diff, and it runs several specialized
reviewer nodes side by side, then has each of their findings independently double-checked
before anything gets reported to you. Nothing here ever changes your code — it only tells you
what it found.

Distinct specialties (each reviewer node owns one concern) plus an otherwise-overloaded
verifier (one reviewer can't credibly judge correctness, security, and style all at once) are
exactly the two signals from `graph-plan`'s Step 0 that justify a graph in the first place —
this skill is what answering "yes" to those looks like, already wired up.

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

Confirm there's an actual diff to look at. If it's trivial — a couple of lines, a config
tweak — say so and suggest a normal read-through instead of spinning up a graph for it. A
graph earns its cost; a two-line change doesn't need one.

## Step 1 — pick the reviewer nodes

Default set: correctness, security, simplification, test coverage. Add or swap based on what
changed (touching auth code pulls in a security-focused node even if not asked; a UI-only
diff might drop test coverage). `--dimensions` overrides this outright. Cap it around five or
six unless `--depth thorough` was asked for — more nodes than that stops adding signal and
starts adding noise and cost.

## Step 2 — run it

Each reviewer node's contract: findings in a consistent shape (file, line, what's wrong, why
it matters), not a paragraph. A reviewer node that doesn't meet its own contract — malformed
output, missing fields — goes through the same repair `graph-run` gives any node: retried with
the specific gap fed back, escalated to `graph-plan` if the same problem recurs, and reported
as blocked or stagnant rather than silently dropped if neither resolves it. Only findings from
a node that met its contract move on.

Every surviving finding then gets checked by a second, independent node — starting clean, none
of the first reviewer's context carried over, only the finding and the diff itself — whose
only job is to try to disprove it. A finding that survives that clean second look is far more
likely to be real than one that was only ever looked at once. One that doesn't survive is
dropped, not repaired — there's nothing to fix about a false positive, it's just not reported.

On `--depth thorough`, add a pass that ranks the surviving findings by severity, and a last
pass that asks what wasn't covered — a file that got skipped, a concern nobody checked — so
nothing is silently left out of the report.

This runs as a background job — you don't have to sit and watch it — but progress is still
narrated live as it happens, the same as any `graph-run` execution: which reviewer nodes have
finished, which are being repaired and why, shown as it happens rather than saved for the
final report. Total cost is bounded by construction — node count (capped around five or six)
times pass count (one by default, up to three on `--depth thorough`) — small enough that a
review this size doesn't need a separate spending cap the way a longer-running, open-ended
graph would.

## Step 3 — report

At default depth, findings come back in the order they were found, each with the evidence
behind it — not just an opinion, but what was actually found and why it matters — and the
report states which dimensions and files were actually checked, so the scope is never left
implicit. On `--depth thorough`, findings are additionally ranked by severity, and a dedicated
pass reports anything a dimension or file didn't get covered. Any reviewer node that came back
blocked or stagnant instead of producing usable findings is named explicitly, not folded
silently into "no findings." Nothing is auto-fixed; you decide what to act on.

## Examples

```
Review my current changes.                        → uncommitted diff, default dimensions
Review PR 42 for security and correctness only.    → --dimensions security,correctness
Do a thorough review of main..HEAD.                → that range, --depth thorough
Show me what a review of this would look like       → --dry-run: lists the planned
  before you actually run it.                         reviewer nodes, runs nothing
```

## What this refuses to do

- Won't grade a finding with the same reviewer node that raised it, or let that check inherit
  the reviewer's own context — the check is always a different pass, starting clean.
- Won't run on a trivial diff just because it was asked to — it'll say so and suggest a plain
  read instead.
- Won't silently skip a file, a concern, or a node that came back blocked or stagnant without
  saying so in the final report.
- Won't go quiet while it runs just because it's a background job.
