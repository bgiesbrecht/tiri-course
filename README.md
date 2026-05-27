# Building Trustworthy Data Agents — A Tiri Field Guide

A self-paced course on building production agentic systems, using
[Tiri](../tiri/) as the working artifact.

This course is **not** a tutorial on how to call an LLM API. It is a
course on how to build an agent that earns trust from a non-technical
audience who will act on its answers. The difference shows up in the
architecture, not just the prompts.

## What you'll build toward

By the end you will have:

- Implemented a parallel, slimmed-down version of Tiri's core pipeline
  on your own dataset.
- Made a benchmark suite pass at 100%, then debugged a regression you
  introduced deliberately.
- Extended the system with one new capability of your choice (a new
  agent, a new provider, a new MCP tool, or a new room from your own
  data).

You won't memorize a framework. You'll internalize a handful of design
rules that survive after frameworks change.

## Audience

Engineers comfortable with Python who want to build agentic systems
that work in production, not toys. **No prior GenAI background is
assumed** — concepts like LLMs, embeddings, retrieval, and prompts
are introduced inline the first time they appear, and gathered in
[Primer.md](./Primer.md) for reference. Deep ML background is not
required. Databricks experience is a bonus, not a prerequisite — the
course can be completed entirely against local providers (DuckDB,
Chroma, SQLite, OpenAI/Anthropic).

> **If you get lost on a term, check [Primer.md](./Primer.md).** It's
> the dedicated reference for every GenAI and Tiri-specific concept
> the course uses. Skim it once before Module 0 to get oriented; come
> back to it whenever something stops making sense.

## The big picture

Before any modules, here is the shape of the system you'll build.
You don't need to understand every box yet. The labs fill in details
in order; this diagram is the map you can return to.

```
            ┌──────────────────────────────┐
            │   User asks a question       │
            │  "revenue by region last Q?" │
            └──────────────┬───────────────┘
                           ↓
   ┌─────────────────────────────────────────────────────┐
   │     CONTEXT ASSEMBLY  (Modules 4–5)                 │
   │     — no reasoning here, just gathering facts       │
   │                                                     │
   │  Catalog     ──→  what tables exist, columns, types │
   │  Metadata    ──→  what tables MEAN (descriptions,   │
   │  (YAML)            synonyms, semantic types)        │
   │  Examples    ──→  pull the top-k most similar       │
   │  (vector idx)      past Q&A pairs                   │
   │                                                     │
   │              produces a ContextPackage              │
   └─────────────────────┬───────────────────────────────┘
                         ↓
   ┌─────────────────────────────────────────────────────┐
   │     AGENT PIPELINE  (Module 6)                      │
   │     — small focused LLM calls, each does one job    │
   │                                                     │
   │   Intent Agent     "answerable? what kind?"         │
   │        ↓                                            │
   │   Plan Agent       "what steps do I need?"          │
   │        ↓                                            │
   │   SQL Agent        generate → VALIDATE → execute    │
   │        ↓                                            │
   │   Synthesis        "explain in plain English,       │
   │   Agent             with evidence and caveats"      │
   │        ↓                                            │
   │   Viz Agent        chart spec  (built in Python,    │
   │                                  no LLM call)       │
   └─────────────────────┬───────────────────────────────┘
                         ↓
   ┌─────────────────────────────────────────────────────┐
   │     Answer surfaced to the user:                    │
   │       • plain-English summary                       │
   │       • the SQL that produced it                    │
   │       • the actual table of results                 │
   │       • what the data does NOT support              │
   │       • confidence level                            │
   └─────────────────────────────────────────────────────┘
```

### What this is doing, in plain English

You ask a question. The system gathers everything it might need to
answer it — schema info, business definitions, similar past examples
— and packages those into one bundle (the **ContextPackage**). Then
a sequence of small, focused **agents** (each one a single LLM call
with a tight job) work on that bundle: classify the question, plan
the steps, write SQL, run it, summarize. The output isn't just an
answer — it's an answer *with evidence and explicit limits*.

The contrast worth holding in your head:

| A naïve "LLM answer this question" approach | What this course builds |
|---|---|
| One giant prompt, one model call | Many small focused calls, each tested |
| Model decides whether SQL is safe | Validator decides, before execution |
| Model invents tables that look plausible | Retrieved metadata is the only source of truth |
| "Here is your answer" | "Here is the answer, the SQL, the rows, and what the data does NOT support" |
| Confidence is the model's tone | Confidence is a structured field with a basis |

## How to take it

- **Self-paced.** No cohort, no deadlines. Most learners will finish in
  6–10 weeks at ~4 hours/week.
- **Read-along + build-along.** Each module pairs reading from the
  Tiri repo with a lab you implement yourself in a separate working
  directory. The labs are graded by their own tests, not by Tiri's.
- **Local or Databricks, your choice.** Every lab specifies both paths.
  If you have a Databricks workspace, modules 7–10 get noticeably more
  interesting; if you don't, the local stack covers everything.

## Prerequisites

Before Module 0, confirm:

1. You have Python 3.11+ and can install dev dependencies.
2. You have an API key for at least one of: OpenAI, Anthropic, or a
   reachable Databricks Model Serving endpoint.
3. You have read access to the Tiri repo as a reference. The course
   does not modify it — it builds in parallel.
4. (Recommended) You have cloned Tiri locally and run
   `pytest tests/unit/` once. If that passes (467+ tests), your env
   is ready.

If step 4 fails, fix that before continuing. Every module assumes the
reference implementation is runnable.

You do **not** need:
- Prior LLM/AI experience (concepts are introduced inline; full
  reference in [Primer.md](./Primer.md)).
- Familiarity with vector databases, embeddings, or RAG.
- Experience with Databricks. (Helpful for Modules 7–10, not required.)

You **do** need:
- Comfort writing Python: dataclasses, abstract classes, async/await,
  pytest. No specific async library beyond what stdlib gives you.
- Comfort reading existing code in another repo for context.

## Module map

| # | Title | Anchor doc in Tiri |
|---|---|---|
| 0 | Frame — why this exists | `docs/vision.md` |
| 1 | Data models as contracts | `docs/data_models.md` |
| 2 | Providers as the I/O boundary | `docs/providers.md` |
| 3 | Configuration & the container | `docs/configuration.md` |
| 4 | Knowledge: metadata + retrieval | `docs/knowledge_store.md`, `docs/metadata.md` |
| 5 | Prompts as files | `tiri/engine/prompt_templates/` |
| 6 | The compound agent pipeline | `docs/agents.md` |
| 7 | Orchestration: the RoomEngine | `docs/room_engine.md` |
| 8 | Surface: REST + SSE streaming | `docs/api.md` |
| 9 | Evaluation as a first-class discipline | `docs/feedback.md`, `docs/tuning.md` |
| 10 | Extensions: composability | `docs/extensions.md` |
| 11 | Capstone | (your choice) |

## How each module is structured

1. **Frame** — the one design rule this module is built around, in one
   sentence.
2. **Read** — files in the Tiri repo to read first, with what to look
   for.
3. **Concepts** — the ideas the module teaches. New GenAI terms get
   short inline definitions in **`> Concept:`** blocks.
4. **Lab** — what you'll build, with a measurable definition of done.
   Most labs are broken into numbered checkpoints with skeleton code
   you fill in.
5. **Common pitfalls** — failure modes that happen often enough that
   it's cheaper to warn you than let you find them.
6. **Stretch** — an optional harder variant.
7. **Reflection** — two or three questions to think through before
   moving on. Not graded; meant to prevent the "I copied the lab and
   moved on without understanding it" failure mode.

## The design rules, in one place

These appear one at a time across the modules. Listed here so you can
return to them.

1. **Vision is the tiebreaker.** When two technically valid choices
   conflict, the system's purpose decides. Tiri is a witness, not an
   analyst.
2. **Foundation before fancy.** Data models are contracts. Invariants
   belong in `__post_init__`, not in the call site that hopes the data
   is right.
3. **Engine has zero I/O.** Anything that talks to the outside world
   goes through a provider. No SDK imports in agents or orchestration.
4. **Configuration is a system, not a dict.** One wiring point. TOML
   plus environment variables. Secrets never in code.
5. **Context is computed fresh per request.** Caching `RoomConfig` is
   how you ship a stale-prompt bug.
6. **Prompts are product surface, not code.** They live in files. They
   have version history. They're not inline f-strings.
7. **Many small focused agents > one omniscient prompt.** Each agent
   does one thing and is testable in isolation.
8. **Validate before execute.** Every SQL string passes a validator
   before it touches the warehouse. No exceptions.
9. **No LLM for structured output you can build deterministically.**
   Vega-Lite specs are constructed in Python, not generated as JSON.
10. **Causal claims are structural, not stylistic.** "Caused by" is
    banned by a regex scan, not by hoping the prompt is good enough.
11. **If you can't measure correctness, you don't have an agent.**
    Benchmarks are the spec. 100% or it's a bug.
12. **Extension beats rewrite.** New capabilities are additive.
    Existing pipelines keep working.

## A note on tone

This course is opinionated. The rules above are not preferences — they
are constraints learned by watching agentic systems fail in production
in ways that erode user trust. You will disagree with some of them.
That's fine. Each module names the antipattern the rule prevents so
you can decide for yourself whether the constraint is worth the cost
in your context.

What is not negotiable is the standard. The bar Tiri sets — a
non-technical user gets a defensible answer with visible evidence and
honest uncertainty — is the bar this course teaches you to meet.
