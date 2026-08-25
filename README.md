# Graph Engineering

A working toolkit for coordinating more than one AI agent on a task — deciding when you
actually need to, and a set of skills that do the coordinating for you.

## Why this exists

Most of the time, one agent doing one job, checking its own work, and trying again when it's
wrong, is all you need. I kept running into tasks where that stopped being true — where one
agent was quietly trying to be a researcher, a writer, and its own reviewer all at once, and
every one of those jobs suffered for it.

Splitting a task like that into several smaller, specialized agents — each with one job, and
clear rules for what gets handed from one to the next — is what I've been calling graph
engineering. This repo isn't a description of that idea. It's a working implementation of
it: skills that actually plan a task into pieces, run those pieces, check each one, repair
what fails, and hand you back a finished result.

## What graph engineering actually is

Picture a task broken into pieces, where each piece is done by its own agent, and there's an
explicit plan for what moves between them. That's the whole idea. Three parts, always:

- **Nodes** — a piece of work with one clear job. "Researcher." "Security reviewer." Not
  "step 3."
- **Edges** — how work moves from node to node. Sometimes that's a judgment call made after
  seeing a node's output; sometimes it's fixed ahead of time and doesn't change no matter
  what the output says. Most edges here point forward; occasionally one points backward —
  send the work back to an earlier node with specific feedback, and let it try again.
- **A contract per node** — what that node must produce, written concretely enough that
  meeting it can actually be checked, by another agent or by a plain mechanical test. "A
  findings list with file, line, issue, and why it matters" is a contract. "Good analysis" is
  not — nobody can tell whether that was met, including the node that produced it.

More agents are more expensive and slower to debug than one good agent, so this isn't the
default. Before splitting anything up, it's worth asking: would a clearer instruction handle
this? Is one step really trying to judge several different things at once, when narrowing it
would fix that without adding more steps? Do the pieces genuinely need different
instructions, tools, or models — or genuine parallelism, or fixed and auditable routing —
and not just "more of the same work" wearing a different label? If none of that's true, a
single well-instructed agent is the right answer, and every skill in this repo checks for
that before doing anything else.

## Loop engineering — the layer this sits on top of

Zoom into any one node here and you'll find something smaller and more familiar: an agent
doing its work, getting checked, and trying again if it's wrong. That's a loop — one agent,
running its own discover-do-verify-retry cycle, on its own. I call the practice of designing
that cycle well "loop engineering": deciding what the agent looks at
before it starts, what "done" actually means, and — the part that's easiest to skip and most
costly to skip — who's allowed to say no to its own work. A loop with no real check is just an
agent nodding at itself, agreeing that whatever it produced must be fine.

A graph doesn't replace any of that. It's several of those loops running together, each one
still doing its own work-check-retry cycle, with an explicit plan for what passes between them
once each one is done. Graph engineering is not a different discipline from loop engineering —
it's what loop engineering turns into once one loop, however well-built, can't hold the whole
job by itself: the work is really several different jobs, or parts of it can genuinely happen
at once, or one loop's check was quietly trying to judge three unrelated things.

Practically, that means: if you already have a solid habit for the smaller loop — a good sense
of what a task actually needs before starting, a real independent check that can say no
instead of rubber-stamping — every node in a graph here is built to reuse exactly that
discipline, just once per node instead of once for the whole task. A graph isn't a different
kind of unit from a loop. It's several of the same kind of unit, coordinated, with the
handoffs between them made explicit instead of left implicit inside one long, overloaded
context window.

## What we built

Two general-purpose tools, a top-level command that wires them together, and seven ready-made
graphs — each one a working example of a genuinely different shape a graph can take, not a
restatement of the same pattern seven times. Nothing here is a description of what a graph
*should* do. Every skill produces a real result: real code that actually runs, real files that
actually get checked, real tests that actually execute — not an agent's opinion about whether
something looks right.

| Skill | What it does | Graph shape it uses |
|---|---|---|
| **`graph`** | The single entry point — plans a task and runs it, in one call. `--auto` for hands-off, `--guided` (default) pauses to confirm the specific uncertain decisions before running. | Wraps the two below |
| **`graph-plan`** | Turns any task into an explicit plan: checks whether it's even needed, then context, nodes, contracts, dependencies. | The planning stage itself |
| **`graph-run`** | Executes a plan (or a task, planning it inline first): nodes in dependency order, each checked, failures repaired and retried without restarting what passed. | The execution stage itself |
| **`graph-review`** | Reviews a diff or PR from several angles at once, each finding double-checked by a skeptic. | Fixed parallel nodes, each reporting independently |
| **`graph-research`** | Researches a question across several angles in parallel, then synthesizes and fact-checks the result. | Parallel nodes feeding into one synthesis node |
| **`graph-content`** | Drafts something, reviews it against a stated bar, and loops back to revise until it passes or stops making progress. | Sequential, with a backward edge |
| **`graph-migrate`** | Applies the same change across a list of files or items, one node per item, checked and repaired individually. | One node per list item, item count decided at plan time |
| **`graph-compare`** | Tries several approaches to the same problem in parallel, then a judge scores and recommends one. | Parallel full attempts feeding into a judge |
| **`graph-build`** | Builds something through a strict ordered chain of stages — scaffold, implement, test, document. | A straight sequential chain, no fan-out, no designed loop-back |
| **`graph-approve`** | Prepares a change fully, fingerprints it, then genuinely pauses mid-execution for a human sign-off before applying anything irreversible. | Sequential, with a real pause built into the middle of it |

## Installing

All of these are Claude Code skills — drop them into `~/.claude/skills/` to make them
available in every project, or into a single project's `.claude/skills/` folder to scope them
to just that project.

```bash
git clone https://github.com/manjeetsc/graph-engineering.git
cp -r graph-engineering/graph          ~/.claude/skills/
cp -r graph-engineering/graph-plan     ~/.claude/skills/
cp -r graph-engineering/graph-run      ~/.claude/skills/
cp -r graph-engineering/graph-review   ~/.claude/skills/
cp -r graph-engineering/graph-research ~/.claude/skills/
cp -r graph-engineering/graph-content  ~/.claude/skills/
cp -r graph-engineering/graph-migrate  ~/.claude/skills/
cp -r graph-engineering/graph-compare  ~/.claude/skills/
cp -r graph-engineering/graph-build    ~/.claude/skills/
cp -r graph-engineering/graph-approve  ~/.claude/skills/
```

Claude Code picks them up automatically after that — there's no command to memorize, just ask
in plain language and the right one loads.

## Examples

**Not sure where to start, or whether you even need this:**

```
Should this be a graph, or is one agent enough?  → graph-plan's Step 0 answers this first
Plan this out before we run anything.             → graph-plan
Just run this end to end.                         → graph, guided mode by default
Run this end to end, auto mode.                    → graph --auto
```

**Reaching for a specific ready-made job:**

```
Review my current changes for bugs, security, and performance.   → graph-review
Research the tradeoffs between these two approaches.             → graph-research
Draft a project update and have someone check it before I see it. → graph-content
Rename this function everywhere it's used in the repo.           → graph-migrate
Try this two different ways and tell me which is better.         → graph-compare
Build this feature: scaffold, implement, test, document.         → graph-build
Prepare the deploy but wait for my okay before pushing it.       → graph-approve
```

**If a skill doesn't trigger on its own,** say so directly: "use graph-review on this." That
always works as a fallback.

## A worked example, start to finish

Say you ask: *"review PR 42 for security and correctness, thoroughly."*

1. `graph-review` loads. It checks first whether this is even worth a graph — several
   independent concerns (security, correctness, plus whatever else the diff touches) that one
   reviewer shouldn't judge all at once — yes, it qualifies. The word "thoroughly" is what
   turns on `--depth thorough` — a ranking pass and a "what got missed" pass, on top of the
   default single pass.
2. It picks the reviewer nodes for this diff (the default set, since nothing narrowed it) and
   shows the plan before running anything:
   ```
   [ ] correctness reviewer
   [ ] security reviewer
   [ ] simplification reviewer
   [ ] test coverage reviewer
   ```
3. All four run in parallel, each reading only the diff and its one concern — the security
   reviewer never sees "check for typos," the correctness reviewer never sees "check for SQL
   injection." Each reports findings in the same shape: file, line, issue, why it matters.
4. Every finding then gets a second look from a fresh pass whose only job is to try to
   disprove it. A finding that survives that is far more likely to be real than one only ever
   looked at once. The checklist updates live as this happens:
   ```
   [✓] correctness reviewer     — passed, 2 findings
   [✓] security reviewer        — passed, 1 finding
   [✓] simplification reviewer  — passed, 0 findings
   [↻] test coverage reviewer   — didn't pass its own check, repairing
   ```
5. Because `--depth thorough` was triggered, two more passes run once the reviewer nodes
   settle: one ranks the surviving findings by severity, one checks for anything no dimension
   or file actually got covered. You get back a ranked report — most serious finding first,
   each with the evidence behind it, plus what the review didn't get to, stated plainly rather
   than left implicit. Nothing in your code changes. You decide what to act on.

That's the same mechanism underneath every skill here — plan, run nodes, check, repair,
report — just pre-configured differently for each job. `graph-content`'s version of step 4 is
a writer getting sent back with specific feedback instead of a finding getting double-checked.
`graph-compare`'s version of step 3 is full independent attempts instead of findings.  Once
you've seen one, you've seen the shape of all of them.

## License

MIT — use it, change it, ship it.
