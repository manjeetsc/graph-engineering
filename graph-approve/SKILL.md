---
name: graph-approve
description: Prepares a change fully — computes exactly what it would do, but doesn't apply it — then genuinely pauses mid-execution for explicit human sign-off, bound to that exact prepared version, before anything irreversible happens. Use when asked to do something that shouldn't happen without a human okaying it first — a deploy, a mass send, a delete, a production change.
---

# Graph Approve

A ready-made `graph-run` configuration for the one thing nothing else in this set does: a real
pause *during execution*, not during planning. Guided mode already asks about genuinely
uncertain points while a plan is being built, then runs straight through once execution
starts — that's the right default for most graphs, but it's wrong for a task where the actual
risk is in the applying, not the planning. This skill exists for that case specifically.

## Arguments

| You say | What it does |
|---|---|
| the change or action | prepares it fully, pauses for sign-off, applies only on approval |
| `--auto-timeout N` | if there's no response after N (minutes/hours you specify), treat it as a rejection, not an approval — silence never defaults to yes |

## Before running

If the action isn't actually irreversible or costly to undo, a plain `graph-run` (or the task
directly) is enough — the pause here exists specifically to protect against something you
can't easily take back. Don't add it as ceremony for something a normal repair cycle would
already catch and fix cheaply.

## Step 1 — prepare, don't apply

A node does all the real work of preparing the change — the diff, the migration, the message
that would be sent, whatever the action actually is — computed in full, checked the normal
way (built and run, for code), but held. Nothing from this step touches anything outside the
prepared artifact itself. If preparing it fails its own check, that's ordinary repair, same as
any node; the pause in Step 3 only ever happens over something that already passed.

## Step 2 — fingerprint what was prepared

Take the exact prepared output from Step 1 and bind it to something simple and checkable — a
content hash is enough, doesn't need to be cryptographic, just specific enough that "this
fingerprint" can only mean "this exact prepared version." This is what stops a stale approval
from silently authorizing something that changed after the fact: if Step 1 ever runs again
(a re-prepare, a retry), the fingerprint changes with it, and any earlier approval no longer
matches anything.

## Step 3 — pause, for real

Show the prepared change and its fingerprint, and stop — not "ask a clarifying question and
keep going," an actual stop, waiting for one of: explicit approval naming the fingerprint,
explicit rejection, or (if `--auto-timeout` was given) silence past the timeout, which counts
as rejection, never as approval. This is the one place in this whole skill set execution
genuinely halts rather than narrating progress while it keeps moving.

## Step 4 — re-check, then apply

On approval: before applying anything, recompute the fingerprint of what's about to be applied
and confirm it still matches what was approved. If nothing changed while waiting, apply exactly
what was shown — no more, nothing extra folded in. If it doesn't match anymore (something
changed the underlying state while the approval was pending), don't apply it — go back to Step
1 and re-prepare against current reality, then pause again for a fresh approval. On rejection
or timeout: apply nothing, and report the prepared-but-unapplied state plainly, so it's clear
work was done and simply not acted on, not that nothing happened.

## Report

What was prepared, its fingerprint, whether it was approved, rejected, or timed out, and — if
applied — confirmation the fingerprint still matched at apply time. If never applied, the
prepared artifact is still reported in full; preparing something is real work and doesn't
disappear just because it wasn't used.

## Examples

```
Prepare tonight's deploy and wait for my okay before pushing it.
Draft the mass email to all customers, but don't send until I approve it,
  --auto-timeout 24h.
Prepare the database migration for review; apply only once I confirm.
```

## What this refuses to do

- Won't apply anything without an approval that names the exact fingerprint of what's being
  applied — a vague "yes, go ahead" isn't enough if it doesn't bind to a specific version.
- Won't treat silence, a timeout, or an ambiguous response as approval — only an explicit,
  fingerprint-matched yes applies anything.
- Won't apply a stale approval against a version that's since changed — it re-prepares and
  asks again instead.
- Won't skip Step 1's own check just because a human will look at it later — a human approving
  a broken change doesn't make the change not broken.
