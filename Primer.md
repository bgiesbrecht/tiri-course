# Primer — concepts and vocabulary

If a term in any module is unfamiliar, look here. This document is a
reference, not a tutorial — read it once at the start to get
oriented, then return when something specific stops making sense.

It is organized by topic, not alphabetically. The order roughly
matches the order concepts appear in the course.

---

## Index

- [How an LLM actually works (just enough)](#how-an-llm-actually-works-just-enough)
  - LLM • prompt • token • context window • temperature
    • hallucination • deterministic at temperature 0
- [Embeddings and retrieval](#embeddings-and-retrieval)
  - embedding • vector • vector index • top-k retrieval • RAG
- [What is an agent (and what isn't)](#what-is-an-agent-and-what-isnt)
  - agent vs chatbot • tool use • function calling
  - compound agent system • single-prompt vs multi-agent
- [Model Context Protocol (MCP)](#model-context-protocol-mcp)
  - MCP server • MCP client
- [Databricks-specific terms](#databricks-specific-terms)
  - Unity Catalog • Genie • Model Serving
    • Vector Search • SQL Warehouse • Statement Execution
- [Tiri-specific terms](#tiri-specific-terms)
  - Room • RoomConfig • ContextPackage • Snippet
    • Example • Provider • Container • Pipeline
    • benchmark • confidence • witness mode • hypothesis mode

---

## How an LLM actually works (just enough)

**LLM (Large Language Model).** A neural network that takes text as
input and produces text as output. GPT-4, Claude, Llama, Gemini are
LLMs. From the outside they look like: "I send a list of messages,
I get back a string." The two facts every engineer needs to know:

1. **They have no memory between calls.** Anything the model should
   know — context, prior turns, instructions — must be in the
   prompt you send.
2. **They produce plausible-sounding output regardless of whether
   it's true.** The model is trained to continue text in a coherent
   style. Fact-checking is your job, not theirs.

**Prompt.** The text you send to the LLM. Usually structured as
messages with roles:

```python
[
    {"role": "system", "content": "You are a SQL generator."},
    {"role": "user", "content": "How many orders did we have?"},
]
```

The `system` message sets the model's persona and constraints. The
`user` (and optionally `assistant`) messages form the conversation.

**Token.** The unit an LLM processes. Roughly: word fragments. "The"
is one token; "anticonstitutional" is several. A 1000-word document
is typically 1300–1500 tokens. You don't usually need to count them
manually, but you do need to know they exist because of:

**Context window.** The maximum number of tokens a model can read
*and* generate in a single call. Modern models have windows from
~8K (small/cheap models) to 200K+ (large/expensive ones). When you
exceed it, you get an error or silent truncation. This is one reason
**retrieval** (below) matters: you can't stuff your entire data
warehouse into a prompt.

**Temperature.** A knob on each LLM call that controls randomness.
`temperature=0.0` makes the model pick the highest-probability token
at each step — output is (mostly) deterministic and reproducible.
Higher values introduce randomness for more "creative" output.

**Deterministic at temperature 0.** Tiri (and this course) uses
`temperature=0.0` everywhere. This is not because creativity is bad;
it's because reproducibility is essential. If the same question
produces a different answer every time, you can't write tests, can't
run benchmarks, can't debug a regression. Determinism is the price
of a system you can verify.

**Hallucination.** When the LLM produces fluent text that's factually
wrong. The model invents a table name that doesn't exist, names a
column that was never defined, quotes a statistic it made up. Most
of this course is shaped by the goal of making hallucinations either
*impossible* (validate SQL against the real schema before running
it) or *visible* (show the evidence; mark confidence honestly).

The term "hallucination" is somewhat softening — it makes the
behavior sound like an honest mistake. A more useful frame: the
model has no internal distinction between "true" and "plausible." It
will produce confidently-stated falsehoods whenever its training
data nudges it in that direction. Your job is to architect around
that.

---

## Embeddings and retrieval

**Embedding.** A way to turn a piece of text into a fixed-length
list of numbers (a "vector"), such that texts with similar *meaning*
produce vectors that are close together. The classic example:

- "How much did we sell?"  →  `[0.12, -0.34, 0.88, ..., -0.05]`
- "What was our revenue?"  →  `[0.13, -0.32, 0.89, ..., -0.04]`
- "What's the weather today?"  →  `[-0.42, 0.71, 0.03, ..., 0.55]`

The first two vectors are close (cosine similarity near 1.0). The
third is far away. The vectors have nothing intrinsically meaningful
about them — the closeness is the only thing that matters.

Embeddings are produced by an embedding model, usually exposed by
the same vendor as the LLM. In the code:

```python
vecs = await llm.embed(["how much did we sell?", "the weather"])
# vecs is a list of two lists of floats, each ~1500 floats long.
```

**Vector.** A list of floating-point numbers. In this course, always
the output of an embedding call. You don't construct them by hand.

**Vector index / vector database.** A data structure that stores
vectors and answers "give me the *k* nearest" queries efficiently.
Naive nearest-neighbor search is O(n) over all vectors; a real
vector index uses approximate algorithms (HNSW, IVF) to do this in
O(log n) for millions of vectors.

Local options: Chroma, FAISS, sqlite-vss. Production options:
Databricks Vector Search, Pinecone, Weaviate. All of them have the
same shape: `upsert(ids, texts, embeddings)` and `query(embedding,
top_k) → list of matches`.

**Top-k retrieval.** "Give me the *k* most similar items to this
query." A standard operation. `top_k=3` returns the three nearest
matches; `top_k=10` returns the ten nearest. You almost always pick
a small k (3–10) because beyond that, results stop being relevant.

**RAG (Retrieval-Augmented Generation).** A pattern: before calling
the LLM, fetch the most relevant context (using vector search or
other lookups) and include it in the prompt. This is how a model
"knows" about your data without being retrained.

The canonical RAG flow:

1. User asks a question.
2. Embed the question.
3. Query the vector index for the top-k most similar documents.
4. Stuff those documents into the LLM's prompt as context.
5. LLM answers using the retrieved context.

Tiri uses RAG for example Q&A pairs — past questions and their
correct SQL. When a new question comes in, the most similar past
examples are pulled and shown to the SQL agent as "here's how
questions like this are typically answered."

The non-RAG alternative — "just put everything in the prompt" — only
works at small scale. RAG is what lets you have a 10,000-table
warehouse and not blow your context window.

---

## What is an agent (and what isn't)

**Agent vs chatbot.** A chatbot replies. An agent *does things*.
Concretely: an agent inspects context, decides what action to take
next (call a tool, ask a clarifying question, write some SQL,
synthesize an answer), executes that action, observes the result,
and may take another step.

In this course, every "agent" is a small unit: one LLM call, one
job, one structured return value. The pipeline of several agents
working in sequence is what produces a real answer. This is the
**compound agent system** pattern.

**Tool use.** When an agent calls something other than the LLM — a
database, an API, another agent, a Python function. The vast
majority of an agent's interesting behavior is tool use. The LLM is
mostly there to *decide which tool to use* and *summarize what came
back*.

**Function calling.** A feature offered by some LLMs (OpenAI's
"function calling", Anthropic's "tool use" API) where the model can
output a structured "call function X with args Y" response instead
of plain text. Useful when you want the model itself to pick from a
menu of tools.

Tiri *mostly does not use function calling*. Instead, each agent is
already specialized for one job, and the engine routes between
agents. The trade-off:

- Function calling: one big model that can do anything; you trust
  it to pick the right tool. Flexible. Hard to test.
- Compound agents: many small specialized models; the engine routes
  deterministically. Less flexible. Each piece is testable in
  isolation.

For high-stakes data systems, the compound-agent route is what wins.
For agents that need to do open-ended things across many domains,
function calling is the right pattern.

**Compound agent system.** The pattern this course teaches. A
sequence of small focused agents, each one a single prompt with a
single job, orchestrated by deterministic code (not by the LLM).
Tiri's pipeline:

```
IntentAgent → PlanningAgent → SQLAgent → SynthesisAgent → VizAgent
```

Each one takes input, makes one LLM call, returns a typed result.
The orchestration code decides what runs when. Module 6 builds
these; Module 7 wires them together.

**Single-prompt vs multi-agent.** The choice you'll make over and
over. The single-prompt approach: one system prompt that tells the
model to "first do X, then Y, then Z, and return the result." The
multi-agent approach: separate prompts and separate calls for X, Y,
Z. The multi-agent route is more code, more LLM calls, more cost —
but each piece is testable, the failure modes are localized, and
you can swap models per task. In production systems where
correctness matters, multi-agent wins.

---

## Model Context Protocol (MCP)

**MCP (Model Context Protocol).** An open standard for exposing
tools and data to AI systems over a uniform interface. Think "REST,
but designed for an agent on the other end."

In practical terms: if you build an MCP server, *any* MCP-aware AI
system (Claude Desktop, Cursor, your own agent) can call its tools
without per-integration plumbing.

**MCP server.** A program that exposes a set of tools (functions
the agent can call) and resources (data the agent can read) over the
MCP protocol. Tiri ships an MCP server so other agents can ask Tiri
questions as a tool.

**MCP client.** The other side — an agent that *calls* MCP servers.
Tiri can also be an MCP client (an "MCPProvider" in our code) so
your agent can pull data from external systems: documentation
servers, ticketing systems, internal search APIs.

MCP doesn't show up until Module 10. If it's not making sense yet,
skip it.

---

## Databricks-specific terms

These appear throughout the course even if you're running locally,
because the production target is Databricks. The course is designed
so you can read these as "this is what the production path looks
like" and continue with local equivalents.

**Unity Catalog (UC).** Databricks' metadata layer. Three-level
namespace: catalog → schema → table. Provides table/column comments,
permissions, lineage. The reference implementation of
`CatalogProvider`. Locally you use a DuckDB-backed equivalent.

**Genie.** Databricks' built-in natural-language-to-SQL feature.
Used in this course as a reference point in the concept map for
readers who've encountered it.

**Model Serving.** Databricks' service for hosting and calling LLMs
(both Databricks-hosted models like Llama, and external models like
GPT and Claude proxied through Databricks). The reference
implementation of `LLMProvider` in Databricks mode.

**Vector Search.** Databricks' managed vector index service.
Production backing for `VectorProvider`. Locally you use Chroma.

**SQL Warehouse.** A cluster optimized for running SQL queries.
The compute behind `QueryProvider.execute()` in Databricks mode.
Locally you use DuckDB.

**Statement Execution API.** The REST API that Tiri uses to run SQL
on a SQL Warehouse. Most of what `DatabricksQueryProvider` is doing
under the hood.

---

## Tiri-specific terms

Vocabulary you'll encounter when reading Tiri's docs and code.

**Room.** A configured instance of the system scoped to a set of
tables, examples, and an audience. Equivalent to a Genie Space. One
deployment of Tiri can host many rooms.

**RoomConfig.** The data shape that defines a Room — table list,
example Q&A pairs, SQL snippets, mandatory filters, the system
prompt's instruction. Stored persistently, reloaded fresh on every
request (rule 5).

**ContextPackage.** The bundle of facts assembled fresh for each
question: table schemas, retrieved examples, conversation history,
joins, business definitions. Every agent in the pipeline takes a
ContextPackage as input. Module 4 builds it.

**Snippet (SQL snippet).** A reusable named SQL fragment that the
room author defines — e.g., `current_quarter_revenue` →
`SUM(o_totalprice) WHERE o_orderdate >= ...`. The SQL agent can use
snippets to assemble queries instead of writing every formula from
scratch.

**Example (Example SQL).** A past question and its correct SQL,
stored in the room. At request time, the top-k most similar examples
are retrieved (via embedding similarity) and shown to the SQL agent
as guidance.

**Provider.** An abstract interface for one category of external
I/O — LLM calls, catalog reads, SQL execution, etc. Defined in
`providers/base.py`, implemented separately for Databricks and
local. Module 2.

**Container.** The single function (`build_container(cfg)`) that
takes a `Config` and returns a dict of provider instances. The only
place provider constructors are called. Module 3.

**Pipeline.** The orchestrated sequence of agents that runs in
response to a question: Intent → (Clarify | SQL) → Synthesis →
Viz. Owned by the `RoomEngine`. Module 7.

**Benchmark.** A stored question with known-correct SQL, used to
verify that a room's agents produce the right answer. Benchmarks are
how you measure correctness. Module 9.

**Confidence.** A structured field on every answer: `high` |
`medium` | `low`. Not a number, not the model's tone — a categorical
field set by the synthesis agent based on whether the data fully
supports the claim, partially supports it, or barely supports it.

**Witness mode (default).** Tiri's default posture. The system
reports what the data shows, with evidence and caveats, but does
*not* make causal claims, predictions, or recommendations. See
`docs/vision.md`.

**Hypothesis mode (opt-in).** A bounded extension where the system
may generate *provisional* explanations clearly marked as
hypotheses, with the data that supports and contradicts each.
Off by default, enabled per-room. See `docs/extensions.md` EXT-11.
