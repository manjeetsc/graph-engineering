---
name: graph-plan
description: Turns a multi-step task into an explicit graph before anything runs. Checks whether the task actually needs more than one agent, and if it does, produces the shared context, a checkable contract for each step, and how the steps depend on each other. Use when asked to plan, scope, or design a multi-agent approach to a task, before running something large or risky, or when asked "how would you break this down" or "should this be a graph." Hands off to graph-run to actually execute.
---

# Graph Plan

Produces one thing: a plan. Not code, not a finished task, a concrete description of what
needs to happen, checkable enough that `graph-run` can execute it without you standing over
it. If you already know the shape of what you want and it's small, you can skip straight to
`graph-run` and let it plan inline. Use this one when the task is bigger, riskier, or you want
to see the shape before anything runs.

## Step 0: check whether this needs a graph at all

Don't skip this. Most tasks don't. Ask:

1. Would a clearer instruction or more context make one agent handle this fine? If yes, stop
   here: say so, and hand back a single-agent recommendation instead of a plan.
2. Is there one reviewer or one step secretly trying to judge several different things at
   once? That's usually a reason to narrow that one step, not to add more of them.
3. Do the steps genuinely need different instructions, tools, or models, not just "more of
   the same work"?
4. Does something need to happen in parallel, with results actually combined afterward?
5. Does the routing between steps need to be fixed and auditable, rather than a judgment call
   each time?

Any real "yes" to 3, 4, or 5 (or a step 2 that can't be fixed by narrowing) is a reason to
keep going. Zero of those means stop: a plan adds cost a single well-instructed agent
wouldn't have needed.

## Step 1: gather the context

Before deciding what the steps are, pin down what's actually true and available for this
task: the files, prior decisions, and constraints every step is allowed to treat as given.
Keep it separate from the task instructions themselves. Vague or missing context here is the
single biggest cause of bad output three steps later, and it's cheaper to fix once, up front,
than to patch after every node inherits the same gap.

Then scope it: not every node needs all of it. A security reviewer doesn't need the writing
style guide; a writer doesn't need the security findings. Give each node only what its own
job actually requires. This isn't just tidiness: a node handed everything will sometimes act
on something it was never meant to weigh in on, and it's harder to tell afterward which fact
actually drove which output.

This context lives for one run. Deciding what a single graph is allowed to treat as true is
this skill's job; remembering something across separate, later runs is a different concern.
If a task needs that, pair a graph with whatever gives a single agent's loop persistent state
across runs (a state file it reads and updates each time it's invoked). Don't try to rebuild
that here; wire the two together instead.

## Step 2: break it into nodes

Each node is one job, named for what it owns ("researcher," "security reviewer," not "step
3."). For each node, decide what it depends on: does it need another node's output before it
can start, or can it start immediately? Nodes with nothing to wait on are what makes
parallel execution possible later; don't invent a dependency that isn't real just to make the
plan feel more sequential and safe.

If two nodes would touch the same file or resource, that's not genuine parallelism even if
neither one is waiting on the other's *output*: it's a real dependency on the resource
itself, and modeling it as parallel invites one node's change to clobber the other's. Either
sequence them, or split the resource so each node owns a distinct piece of it. Don't discover
this collision after the fact; decide ownership here, at planning time.

## Step 3: write a contract for each node

A contract says what the node must produce, in a shape that can be checked without a person
reading it and forming an opinion. "A findings list with file, line, issue, and why it
matters" is a contract. "Good analysis" is not: nobody, including the node that produces it,
can tell if that was met. If you can't write a checkable contract for a node, that's usually a
sign the node's job is still too vague; narrow it until you can.

## Step 4: decide how each contract gets checked

Two options, per node:

- **A second pass**: a fresh node whose only job is to try to disprove the first node's
  output against its contract. Use this when meeting the contract is a judgment call (is this
  finding real, is this claim actually supported).
- **A deterministic check**: does the output literally contain what the contract asked for
  (the required fields, a minimum count, a specific format). Use this when meeting the
  contract is just a fact you can check mechanically. Cheaper, and prefer it wherever it
  actually applies; save the second-pass check for where judgment is truly required.

If a node produces or changes code, its check is a deterministic one by default, and it means
actually running the thing (build it, execute it, run the test suite), not reading the diff
and judging whether it looks right. A second pass that only reads code is still a judgment
call standing in for a fact you could have just checked; run it whenever running it is
possible at all.

Never let a node check its own output. The check is always a different pass than the one that
produced the work.

## Step 5: decide how the pieces come together

State, in one line, how the outputs of the passing nodes combine into the final deliverable:
concatenated, one node synthesizes the others, or something else. If you can't state this in
one line, the node breakdown from Step 2 probably needs another look.

## Step 6: plan one more check, on the assembled whole

Every node passing its own contract doesn't mean the combined result is actually right:
pieces that are each individually correct can still conflict once they're put together (code
that each file's own check passed still might not build once combined; a brief built from
solid research can still misrepresent it in synthesis). Decide now what checks the *assembled*
deliverable, not just its parts: for code, that's building and running the full test suite
against the combined result; for anything else, it's whatever check on the finished whole
would catch a problem no single node's contract could see. This is a distinct check, in
addition to, never instead of, the per-node checks from Step 4. It runs every time the
pieces are assembled, which can be more than once: a failure here sends the responsible
node(s) back through repair and reassembly, same as any other failed check, until it passes
or stops converging.

## Output

A plan is: the scoped context from Step 1, the node list from Step 2 (with ownership settled
for anything nodes might collide on) with each one's contract (Step 3) and check (Step 4), the
assembly rule from Step 5, and the whole-result check from Step 6. State it as a short,
readable checklist (one line per node, in order, plus the final check), not just as data.
Hand this to `graph-run` to actually execute it: say so explicitly rather than leaving the
plan sitting unused.

## Saving a plan for reuse

A plan doesn't have to be re-derived every time. If this exact shape is genuinely going to run
again (the same job repeated on a schedule, or handed off to a different session or a
different runtime), save it as a plain file with everything from Output above: context,
nodes, contracts, checks, assembly rule. `graph-run` can load a saved plan directly instead of
planning inline. This is worth doing when the shape will actually repeat; for a one-off task,
saving it is just a file nobody reopens. Don't do it reflexively just because you can.

## Refuses to

- Skip Step 0 because the task sounds like it obviously needs multiple agents. It's checked
  every time.
- Name two nodes for what's actually one job split for no reason: that's cost with nothing
  behind it.
- Write a contract vague enough that checking it would require a person's opinion every
  single time, when a concrete one was possible.
- Leave a resource collision between parallel nodes unresolved because neither node
  technically depends on the other's output.
- Skip Step 6 because every node already has its own check: a whole that's never checked as
  a whole is a real gap, not redundant caution.
- Save a plan to a file reflexively, for a task that's only going to run once. That's the
  same "premature graph" mistake one level up.
