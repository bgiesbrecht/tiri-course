# Module 11 — Capstone

> **The standard:** a thing you'd be willing to demo to someone
> with real expectations.

You've finished the course. By now you have a working agent that
answers questions from your own data, refuses what it can't defend,
streams events to a client, scores 100% on a benchmark suite, and
exposes itself as an MCP tool. That's the floor.

This module is the ceiling. You'll pick one project, build it to
the standard of the rest of the course (tests, benchmarks,
documented behavior), and ship it.

There is no skeleton code. There is no "expected output." The
work is yours; the goal is to discover whether the discipline of
the prior ten modules has become muscle memory yet.

## Pick one (or propose your own)

### Option A — A real room on your real data

Build a `RoomConfig` from a dataset that actually matters to you.
Not TPC-H. Not your sample. A real internal dataset, a public
dataset you care about, or a domain you've wanted to query for a
while.

The work:

- Pick the audience and the question (return to your Module 0
  manifesto; refine if needed).
- Build the metadata YAML, the example index, the snippets, the
  mandatory filters.
- Author 15–25 benchmarks. Variety: simple aggregations, filtered
  questions, multi-table joins, edge cases, refusals.
- Tune until the benchmark score is 100%.
- Run it for a week against questions you actually have. Track
  what works, what doesn't, what regressed.

Why this is hard: real data has business logic the room author
needs to encode. The first time you see "revenue means SUM(amount)
where status != 'refunded'" appear in a snippet, you'll understand
why Tiri's vision document spends as much time as it does on
audience and trust.

Definition of done: a one-page writeup in your repo
(`docs/learnings.md`) on (a) what made benchmark tuning hardest, (b)
one thing you'd do differently if you started over, (c) what your
audience could ask the agent that it now answers well, and what it
still cannot.

### Option B — A new extension of your own design

Pick a capability the course doesn't cover. Build it as an
*extension* — additive, behind a flag if necessary, with existing
code unchanged.

Examples that have come up in real Tiri-style projects:

- **Cost-aware routing.** The `RouterLLMProvider` chooses a backend
  based on a per-request budget (cheap if the question looks easy;
  smart if it looks hard). Define "looks easy."
- **Scheduled summaries.** A cron job that runs a question every
  morning and posts the answer to Slack. Re-uses the engine; adds
  a scheduler and a delivery integration.
- **An anomaly callout agent.** Watches a small set of metrics; if
  one moves outside a threshold, generates a question, runs it
  through the agent, posts the answer. Same engine; new trigger.
- **A "fact-check this claim" mode.** User submits a sentence
  ("Q3 revenue grew 14%"); the agent generates SQL to verify the
  claim, runs it, returns "supported by the data" / "contradicted"
  / "the data does not address this."
- **A glossary auto-populator.** Watches the questions users ask;
  surfaces terms that appear repeatedly but aren't in the metadata
  YAML. Helps a room author find the gaps in their semantic layer.

Definition of done: working code, tests, at least three
benchmarks (or behavioral tests) that exercise the new capability,
a one-page writeup of the design.

### Option C — A Genie Space → Tiri `RoomConfig` translator

A config-translation exercise. Read a Databricks Genie Space export
and emit a Tiri `RoomConfig` JSON. Useful if you've worked with
Genie Spaces and want a concrete way to see how the two
configuration shapes carry the same kinds of information.

The work:

- Read the Genie export format.
- Translate tables, instructions, certified queries, glossary terms
  into Tiri's `RoomConfig` shape.
- Document where the two configurations don't map cleanly in either
  direction — concepts one shape has that the other doesn't.

Definition of done: a working CLI, a translation report on at least
one Space, a one-page document on what didn't translate cleanly.

### Option D — Multi-provider tuning case study

Run your existing benchmark suite against three LLM backends —
e.g., OpenAI, Anthropic, and a local Ollama model. Compare scores.
For each *failure unique to one model*, diagnose the root cause:
prompt issue, missing example, model-specific quirk.

Produce a write-up:

- Total scores per model.
- Per-benchmark pass/fail table.
- For each model, the top three benchmarks it gets wrong that the
  others get right.
- A recommendation: for *your* domain, which model should the room
  default to?

This option is less code than the others; the value is the rigor.
Tiri's own `CLAUDE.md` documents a similar comparison across five
backends and ~10 fixmes that came out of it. Yours can be smaller.

Definition of done: a `docs/model-comparison.md` report with the
above sections.

## Quality bar

A capstone is done when:

1. **It runs.** Someone else with your repo and an API key can
   clone, install, and execute it end-to-end. Document this in
   `README.md`.

2. **It is tested.** Unit tests for code, benchmarks for behavior.
   The boundary you set in Module 6 still holds: tests cover
   plumbing; benchmarks cover quality.

3. **It is honest.** When it fails, it fails *visibly* — error
   Turns, structured messages, no silent corruption. The witness
   discipline you internalized in Module 0 still applies.

4. **It has a write-up.** One page. What you built, why, what
   surprised you, what you'd do differently. Future-you reading
   this in a year should be able to remember the project from the
   write-up alone.

You'll know it's done when you're willing to demo it to someone
who would actually use it.

## Submission

There is no submission. The artifact is the artifact.

If you want feedback:

- Open a PR on a friend's fork. Have them code-review your room or
  your extension. A code review from someone who hasn't taken the
  course is the best signal that your architecture survived a
  cold read.
- Post the write-up somewhere public. The course community will
  surface anyone working on similar problems.
- Run the agent against five questions from someone in your target
  audience who has *not* helped you build it. Their reactions —
  what they trusted, what they doubted, what they ignored — are
  the real evaluation.

## What to take away from the course

If you remember nothing else, remember these:

1. **The architecture is the trust.** The decisions that made the
   agent trustworthy — zero I/O in the engine, validate before
   execute, prompts as files, structural enforcement of causal-
   language bans — were not optimizations. They were the product.

2. **Benchmarks are the spec.** A behavior without a benchmark is
   a behavior that will regress. A score below 100% is a bug
   you haven't isolated.

3. **Decomposition is the lever.** One big prompt is tempting and
   sometimes works. For a system that has to be defensible, many
   small focused agents always wins. The cost is more code; the
   benefit is testability, debuggability, and per-task model
   choice.

4. **Refusal is a feature.** A system that says "I don't know"
   honestly is more useful than one that produces fluent guesses.
   For a high-stakes audience, the second one is worse than
   useless.

5. **Extension beats rewrite.** Every capability worth adding to a
   working agent should be additive. If you find yourself wanting
   to rewrite, look harder — there's usually an extension hiding
   inside the rewrite that gets you what you want for one-tenth
   the cost.

These are not opinions you should hold dogmatically. They are
defaults that have served you in this course; they may not serve
you in every domain. The right move when you find a place they
don't apply is to *write down what changed* and decide what your
new defaults are. The discipline you built here is the discipline
of explicit choice — not following rules from a book.

## A note on what's next

Tiri is one architecture. The patterns generalize, but the
implementations don't. When you build your next agent:

- The shape of `ContextPackage` will be different. The discipline
  of "fresh per request, computed not cached" stays.
- The agents will be different. The discipline of "small, focused,
  testable in isolation" stays.
- The benchmarks will be different. The discipline of "100% or
  it's a bug" stays.

The course gives you the patterns. The work makes them yours.

Go build something defensible.
