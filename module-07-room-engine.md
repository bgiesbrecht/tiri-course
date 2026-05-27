# Module 7 — Orchestration: the RoomEngine

> **The rule:** RoomConfig reloads every request. Degrade, don't
> crash. Orchestration is policy, not just plumbing.

In this module you'll build the `RoomEngine` — the single entry
point that ties context assembly, agents, providers, and persistence
into one coherent pipeline. By the end, you'll have a working
`engine.chat()` method that takes a question and returns a complete
`Turn` (with SQL, results, and an answer), running against your own
DuckDB data and a real LLM.

This is the first module where the system *works* in the end-to-end
sense. Up to now you've been building pieces that test in isolation.
Today they start producing answers.

## Read

1. **`tiri/docs/room_engine.md`** — the whole thing. Focus on:
   - The pipeline diagram (Intent → branch → SQL → Viz → Synthesis).
   - The error-handling rules: each stage's failures degrade
     cleanly, never crash the engine.
   - The rule that `RoomConfig` reloads on every request.

2. **`tiri/tiri/engine/room_engine.py`** — read top to bottom. It's
   ~900 lines but most are helpers. Focus on:
   - `RoomEngine.chat()` (around line 155). This is the spine.
   - `_route_intent()` (around line 406). The branching logic after
     the IntentAgent runs.
   - `_load_room_config()` and the comment about not caching it.
   - `_persist_turn()` and how the engine separates "produce an
     answer" from "save the answer" — both have to succeed for the
     request to be considered done.

3. **`tiri/tiri/engine/room_engine.py`** again, this time the
   `stream_chat()` method (around line 205). Same pipeline, but
   emits intermediate events to the caller. We won't build streaming
   in this module — Module 8 does — but seeing the shape now is
   useful.

## Concepts

### What an engine is, and what it isn't

The engine, in this course, is **the single function that runs the
pipeline**. It's not:

- A framework
- A request handler
- A class hierarchy
- A state machine

It's a method that takes a question and returns a `Turn`. Inside,
it does the steps in order, handles errors, and persists the result.
That's it.

The temptation, especially in Python, is to make this object-y: a
`Pipeline` class with a `register_step()` method and a chain-of-
responsibility pattern. **Don't.** The pipeline has a fixed shape;
the steps are known at design time; the branching is small. A 200-
line method with clear sections is more debuggable than a 50-line
method backed by a configurable pipeline framework.

The flexibility you'd buy with a framework — "I can plug new steps
in without modifying the engine" — is almost never the thing you
need. The thing you need is to debug why one specific question
returned the wrong answer, and a linear method is much easier to
trace.

### Orchestration vs choreography

In a multi-agent system, the question "who decides what happens
next" has two answers:

- **Orchestration:** a central coordinator (the engine) decides
  which agent to call when, in what order, with what inputs. Agents
  are tools the engine uses.

- **Choreography:** each agent decides on its own who to call next
  based on its output. The flow emerges from agents talking to each
  other.

This course is firmly orchestration. The engine's job is to make
the flow legible and testable. The agents' job is to do their small
piece well. The benefit: when something goes wrong, you can read
the engine top to bottom and see exactly what happened. With
choreography, the flow lives inside the agents themselves, scattered
across the codebase.

Choreography has its place — it's a better fit for fan-out, fan-in
workloads where agents work on independent subproblems. For a
question-answering pipeline where the steps are sequential and
correctness matters, orchestration is the right choice.

> **Concept: Compound agent system (revisited).** In the
> orchestration model, a "compound agent system" is exactly: one
> engine that calls many specialized agents. The engine is *not* an
> agent. It's the coordinator. The distinction matters because the
> engine doesn't make LLM calls itself — every LLM call goes through
> one of the agents it owns.

### Why RoomConfig reloads every request

A `RoomConfig` is the configuration that defines a room: which
tables, which examples, which mandatory filters, which instruction
to put in the system prompt. The room author edits it from a UI or
an API call.

The temptation: load it once at engine startup, hold it in memory,
serve from cache.

Why this is wrong:

1. **A room author tunes their room and expects the next question to
   reflect their change.** With cached config, they wait for a
   process restart. They will not.

2. **A bug in the SQL agent gets fixed by adding a snippet to the
   room config.** That fix should land for the next request, not the
   next deploy.

3. **Multi-tenant deployments serve hundreds of rooms.** Caching all
   of them is memory waste; caching some is invalidation hell.

The rule: load from the `StoreProvider` at the start of every
`chat()` call. Cost: one read per request, usually a few ms. Benefit:
the room author's mental model ("I changed it, it changed") is
correct.

The thing you *can* cache, with a TTL: physical schema reads from
the catalog. The catalog doesn't change every 30 seconds, but
re-reading it on every request is wasteful. Most of the pipeline's
freshness is in the room config, not the schema.

### Degrade, don't crash

The pipeline has many failure modes:

- The LLM call times out.
- The intent agent returns malformed JSON.
- The SQL agent exhausts retries.
- The SQL validates but execution fails (a permission error, a
  resource limit).
- The vector index is down.
- The synthesis agent's regex scan finds a banned phrase.

A naive engine catches each of these and crashes. The user gets a
500 error. The room author has no idea what happened.

A good engine *degrades*: it produces a useful `Turn` even when
something fails, with an error field that explains what went wrong.
The user gets:

- For an LLM timeout: "I couldn't reach the language model. Please
  try again."
- For malformed intent JSON: "I couldn't classify your question.
  Please try rephrasing it."
- For exhausted SQL retries: "I generated SQL but it failed to
  validate against the schema after 3 attempts. The last error was:
  ..."

Each of these is a valid `Turn` with the `error` field set. The
engine persists it like any other turn so the room author can
review what went wrong. The pipeline never crashes.

The discipline: every stage of the pipeline is wrapped in a try
that converts unexpected errors into a structured `Turn`. Not
because we're nervous, but because honest error reporting is part
of what makes the system trustworthy.

### Conversation history (just enough)

A real engine threads conversation history through the pipeline so
follow-up questions work:

```
User: What was Q3 revenue?
Agent: $1.2M.
User: How about by region?       ← needs to know "Q3" from prior turn
```

The mechanism: persist each `Turn` keyed by a `conversation_id`. On
the next request with the same id, load the recent history and pass
it to the context builder. The context builder includes the last N
turns in the `ContextPackage`. The agents see the history in their
prompts and can reference prior questions and answers.

We won't fully implement history in this module — it's a small but
fiddly addition. The lab leaves a hook for it and Module 8 wires
it through the API.

## Lab

You'll build a slim `RoomEngine` that ties everything together. Six
checkpoints. Plan for a long session.

### Checkpoint 1 — A minimal `RoomConfig`

In `agent/data_models.py`, add:

```python
@dataclass
class RoomConfig:
    """Defines a room: tables, examples, instructions.

    The room author edits this; the engine reloads it per request.
    """
    room_id: str
    title: str
    tables: list[str]                      # FQNs like "main.orders"
    text_instruction: str = ""             # appended to system prompts
    example_questions: list[str] = field(default_factory=list)
    refused_claim_types: list[str] = field(default_factory=list)
```

Real room configs (Tiri's) have many more fields — snippets, joins,
mandatory filters, hypothesis-mode toggles, MCP server URLs. Keep
it slim for the lab; grow it later.

### Checkpoint 2 — A `StoreProvider` for room configs

Add a simple file-backed store. `agent/providers/file_store.py`:

```python
"""File-backed StoreProvider. JSON-on-disk. Good for local dev."""
from __future__ import annotations
import json
from dataclasses import asdict
from pathlib import Path

from agent.data_models import RoomConfig
from agent.providers.base import StoreProvider, StoreProviderError


class FileStoreProvider(StoreProvider):
    def __init__(self, root_dir: str) -> None:
        self._root = Path(root_dir)
        self._root.mkdir(parents=True, exist_ok=True)

    async def get_room_config(self, room_id: str) -> RoomConfig:
        path = self._root / f"{room_id}.json"
        if not path.exists():
            raise StoreProviderError(f"room not found: {room_id!r}")
        raw = json.loads(path.read_text())
        return RoomConfig(**raw)

    async def put_room_config(self, config: RoomConfig) -> None:
        path = self._root / f"{config.room_id}.json"
        path.write_text(json.dumps(asdict(config), indent=2))
```

Add the abstract base to `agent/providers/base.py`:

```python
# ─── StoreProvider ──────────────────────────────────────────────────

class StoreProviderError(ProviderError): ...


class StoreProvider(ABC):
    """Persistence for room configs and conversation history."""

    @abstractmethod
    async def get_room_config(self, room_id: str) -> "RoomConfig":
        ...

    @abstractmethod
    async def put_room_config(self, config: "RoomConfig") -> None:
        ...
```

Wire it through `agent/container.py`:

```python
def _build_store(cfg: Config) -> StoreProvider:
    from agent.providers.file_store import FileStoreProvider
    return FileStoreProvider(root_dir=cfg.store_dir)
```

And add `store_dir: str = "./rooms"` to your `Config` dataclass. Add
`"store": _build_store(cfg)` to the container dict.

### Checkpoint 3 — Author a room

Create `~/my-agent/rooms/sales-room.json`:

```json
{
  "room_id": "sales-room",
  "title": "Sales Analysis",
  "tables": ["main.orders"],
  "text_instruction": "Revenue is calculated as SUM(amount). Use the orders table for revenue questions.",
  "example_questions": [
    "What was total revenue by region?",
    "Which customer ordered the most last quarter?",
    "Show me average order value by month."
  ],
  "refused_claim_types": [
    "Causes of customer behavior",
    "Predictions of future revenue",
    "Recommendations about pricing or strategy"
  ]
}
```

This room's text_instruction encodes a piece of business logic — how
revenue is calculated. The room author owns this. The engine reads
it at request time and the agents see it in their prompts.

Verify:

```python
# scripts/load_room.py
import asyncio, json
from agent.config import Config
from agent.container import build_container
from agent.data_models import RoomConfig

async def main():
    cfg = Config.load()
    container = build_container(cfg)
    store = container["store"]
    with open("rooms/sales-room.json") as f:
        config = RoomConfig(**json.load(f))
    await store.put_room_config(config)
    loaded = await store.get_room_config("sales-room")
    print(f"Loaded room: {loaded.title}, {len(loaded.tables)} tables")

asyncio.run(main())
```

Run:

```bash
python scripts/load_room.py
```

Expected:

```
Loaded room: Sales Analysis, 1 tables
```

### Checkpoint 4 — The RoomEngine class

`agent/engine/room_engine.py`:

```python
"""RoomEngine — the single entry point that runs the pipeline."""
from __future__ import annotations
import logging
import time
import uuid

from agent.data_models import (
    Answer, ContextPackage, Evidence, IntentResult,
    Question, RoomConfig, SQLResult, Turn,
)
from agent.providers.base import (
    CatalogProvider, LLMProvider, MetadataProvider, QueryProvider,
    StoreProvider, VectorProvider,
    LLMProviderError, QueryProviderError, StoreProviderError,
)
from agent.engine.agents.intent_agent import IntentAgent
from agent.engine.agents.sql_agent import SQLAgent
from agent.knowledge.context_builder import ContextBuilder
from agent.knowledge.example_indexer import ExampleIndexer


_log = logging.getLogger("agent.engine")


class RoomEngineError(Exception):
    """Raised only for unrecoverable bugs in the engine itself.

    Agent or provider errors are converted to error-Turns and not
    raised. This exception is for things like 'the engine was
    constructed without a required provider' — bugs, not bad inputs.
    """


class RoomEngine:
    def __init__(
        self,
        catalog: CatalogProvider,
        metadata_providers: list[MetadataProvider],
        llm: LLMProvider,
        query: QueryProvider,
        vector: VectorProvider,
        store: StoreProvider,
    ) -> None:
        self._catalog = catalog
        self._metadata_providers = metadata_providers
        self._llm = llm
        self._query = query
        self._vector = vector
        self._store = store

    async def chat(
        self,
        room_id: str,
        conversation_id: str,
        question_text: str,
    ) -> Turn:
        """Run the pipeline for one question. Always returns a Turn."""
        started = time.monotonic()
        question = Question(
            question_id=str(uuid.uuid4()),
            text=question_text,
            asked_at=_now_utc(),
        )
        turn_id = str(uuid.uuid4())

        # 1) Reload room config. Stale config is a stale-prompt bug.
        try:
            config = await self._store.get_room_config(room_id)
        except StoreProviderError as e:
            return _error_turn(turn_id, question, f"Room not found: {e}")

        # 2) Build the ContextPackage fresh.
        try:
            context = await self._build_context(question_text, config)
        except Exception as e:
            _log.exception("context build failed for room %s", room_id)
            return _error_turn(
                turn_id, question, f"Failed to assemble context: {e}"
            )

        # 3) IntentAgent — classify.
        try:
            intent = await IntentAgent(self._llm).run(question_text, context)
        except LLMProviderError as e:
            return _error_turn(
                turn_id, question,
                f"I couldn't classify your question. {e}"
            )

        # 4) Branch on intent.
        if intent.kind == "clarification_needed":
            return Turn(
                turn_id=turn_id,
                question=question,
                clarification_question=intent.clarification or (
                    "Could you provide more detail?"
                ),
            )
        if intent.kind == "refused":
            return Turn(
                turn_id=turn_id,
                question=question,
                unable_to_answer=intent.refusal_reason or (
                    "I'm not able to answer that with the available data."
                ),
            )

        # kind == "answerable" — proceed to SQL.
        try:
            sql_result = await SQLAgent(self._llm, self._query).run(
                question_text, context, intent
            )
        except LLMProviderError as e:
            return _error_turn(
                turn_id, question,
                f"I couldn't generate SQL. {e}"
            )

        if not sql_result.is_valid:
            return _error_turn(
                turn_id, question,
                f"I couldn't produce valid SQL. {sql_result.error}"
            )

        # 5) Execute.
        try:
            query_result = await self._query.execute(sql_result.sql)
        except QueryProviderError as e:
            return _error_turn(
                turn_id, question,
                f"SQL validated but execution failed: {e}"
            )

        # 6) Synthesize — for now, a one-line summary.
        #    Module 9 expands this into a proper SynthesisAgent.
        summary = _synthesize_one_line(query_result)
        return Turn(
            turn_id=turn_id,
            question=question,
            answer=Answer(
                text=summary,
                evidence=[Evidence(
                    source=", ".join(intent.relevant_tables),
                    excerpt=f"{query_result.row_count} rows returned",
                    confidence="high" if query_result.row_count > 0 else "low",
                )],
                caveats=_default_caveats(config),
            ),
        )

    async def _build_context(
        self, question: str, config: RoomConfig
    ) -> ContextPackage:
        builder = ContextBuilder(
            catalog=self._catalog,
            metadata_providers=self._metadata_providers,
            llm=self._llm,
            vector=self._vector,
        )
        return await builder.build(
            question=question, tables=config.tables, top_k_examples=3
        )


# ─── helpers ────────────────────────────────────────────────────────

def _now_utc():
    from datetime import datetime, timezone
    return datetime.now(timezone.utc)


def _error_turn(turn_id, question, message) -> Turn:
    """Build a Turn that reports a pipeline failure cleanly."""
    return Turn(
        turn_id=turn_id,
        question=question,
        unable_to_answer=message,
    )


def _synthesize_one_line(query_result) -> str:
    """Stand-in for the SynthesisAgent (Module 9 builds the real one)."""
    if query_result.row_count == 0:
        return "The query returned no rows."
    if query_result.row_count == 1 and len(query_result.columns) == 1:
        return f"{query_result.columns[0]}: {query_result.rows[0][0]}"
    return (
        f"{query_result.row_count} rows across "
        f"{len(query_result.columns)} columns: "
        f"{', '.join(query_result.columns)}."
    )


def _default_caveats(config: RoomConfig) -> list[str]:
    """Make the refusal list visible in every answer."""
    if not config.refused_claim_types:
        return []
    return [f"This room will not produce: {c}." for c in config.refused_claim_types[:2]]
```

Read this method top to bottom. Notice:

- Six numbered steps.
- Each step has its own try/except converting failures to error
  Turns.
- The engine never raises (except `RoomEngineError` for true bugs).
- The synthesis is a stand-in — Module 9 builds the real
  `SynthesisAgent`. For now, a deterministic one-liner that reads
  the query result.

### Checkpoint 5 — Wire the engine into the container

Update `agent/container.py`:

```python
def build_container(cfg: Config) -> dict[str, Any]:
    container: dict[str, Any] = {
        "llm": _build_llm(cfg),
        "query": _build_query(cfg),
        "catalog": _build_catalog(cfg),
        "vector": _build_vector(cfg),
        "store": _build_store(cfg),
        "metadata_providers": _build_metadata_providers(cfg),
    }
    from agent.engine.room_engine import RoomEngine
    container["engine"] = RoomEngine(
        catalog=container["catalog"],
        metadata_providers=container["metadata_providers"],
        llm=container["llm"],
        query=container["query"],
        vector=container["vector"],
        store=container["store"],
    )
    return container


def _build_catalog(cfg: Config) -> CatalogProvider:
    from agent.providers.duckdb_catalog import DuckDBCatalogProvider
    return DuckDBCatalogProvider(data_dir=cfg.query_data_dir)


def _build_vector(cfg: Config) -> VectorProvider:
    from agent.providers.chroma_vector import ChromaVectorProvider
    return ChromaVectorProvider(
        path=cfg.vector_db_path, collection=cfg.chroma_collection
    )


def _build_metadata_providers(cfg: Config) -> list:
    from agent.providers.yaml_metadata import YAMLMetadataProvider
    if not cfg.metadata_yaml_path:
        return []
    return [YAMLMetadataProvider(cfg.metadata_yaml_path)]
```

Add `metadata_yaml_path: str = ""` to `Config`.

You also need to seed the example index for your room. Add to
`scripts/load_room.py`:

```python
# After loading the room config, seed the example index.
from agent.providers.chroma_vector import ChromaVectorProvider
from agent.data_models import ExampleSQL

vec = container["vector"]
llm = container["llm"]
indexer = ExampleIndexer(llm, vec)

# Pair each example_question with the SQL you'd expect for it.
# These are guesses; you can refine them later when benchmarks
# (Module 9) tell you which ones are wrong.
examples = [
    ExampleSQL(
        question="What was total revenue by region?",
        sql="SELECT region, SUM(amount) FROM orders GROUP BY region",
    ),
    ExampleSQL(
        question="Which customer ordered the most last quarter?",
        sql=(
            "SELECT customer_id, COUNT(*) AS n FROM orders "
            "GROUP BY customer_id ORDER BY n DESC LIMIT 1"
        ),
    ),
    ExampleSQL(
        question="Show me average order value by month.",
        sql=(
            "SELECT date_trunc('month', asked_at) AS month, "
            "AVG(amount) FROM orders GROUP BY 1 ORDER BY 1"
        ),
    ),
]
await indexer.index(examples)
print(f"Indexed {len(examples)} examples")
```

Re-run:

```bash
python scripts/load_room.py
```

### Checkpoint 6 — Ask a question

Create `scripts/ask.py`:

```python
"""Run the engine for a single question. Prints the Turn."""
import asyncio
import sys
from agent.config import Config
from agent.container import build_container


async def main(room_id: str, question: str):
    cfg = Config.load()
    container = build_container(cfg)
    engine = container["engine"]
    turn = await engine.chat(
        room_id=room_id,
        conversation_id="manual-001",
        question_text=question,
    )

    print(f"\n── Question ───────────────────────")
    print(turn.question.text)
    if turn.answer:
        print(f"\n── Answer ─────────────────────────")
        print(turn.answer.text)
        if turn.answer.evidence:
            print(f"\n── Evidence ───────────────────────")
            for ev in turn.answer.evidence:
                print(f"  {ev.source}: {ev.excerpt} (confidence: {ev.confidence})")
        if turn.answer.caveats:
            print(f"\n── Caveats ────────────────────────")
            for c in turn.answer.caveats:
                print(f"  - {c}")
    elif turn.unable_to_answer:
        print(f"\n── Unable to answer ───────────────")
        print(turn.unable_to_answer)
    elif turn.clarification_question:
        print(f"\n── Clarification needed ───────────")
        print(turn.clarification_question)


if __name__ == "__main__":
    room_id, question = sys.argv[1], sys.argv[2]
    asyncio.run(main(room_id, question))
```

Run:

```bash
python scripts/ask.py sales-room "What was total revenue by region?"
```

Expected output (with some variation based on your model):

```
── Question ───────────────────────
What was total revenue by region?

── Answer ─────────────────────────
4 rows across 2 columns: region, sum(amount).

── Evidence ───────────────────────
  main.orders: 4 rows returned (confidence: high)

── Caveats ────────────────────────
  - This room will not produce: Causes of customer behavior.
  - This room will not produce: Predictions of future revenue.
```

Your first end-to-end answer. The pipeline ran: room loaded, context
built, intent classified, SQL generated and validated against your
DuckDB schema, SQL executed, results summarized.

Try a refused question:

```bash
python scripts/ask.py sales-room "Why is revenue declining?"
```

Expected (with variation):

```
── Question ───────────────────────
Why is revenue declining?

── Unable to answer ───────────────
I can't determine causes of revenue changes from the data available.
This would require <data the agent doesn't have>.
```

And a clarification-needed question:

```bash
python scripts/ask.py sales-room "tell me about sales"
```

Expected:

```
── Clarification needed ───────────
What would you like to know about sales? For example: total revenue
by region, top customers, or average order value?
```

These three behaviors — answer, refuse, clarify — are the system
working as designed. Each is a `Turn` with exactly one of the three
fields set (Module 1's invariant earns its keep here).

### Definition of done

- `agent/engine/room_engine.py` defines `RoomEngine` with a `chat()`
  method that returns a complete `Turn`.
- `agent/data_models.py` has `RoomConfig`.
- `agent/providers/file_store.py` persists `RoomConfig` to disk.
- `scripts/load_room.py` loads your sample room and seeds examples.
- `scripts/ask.py` answers at least three questions from your room
  (one answerable, one refused, one needing clarification) without
  the pipeline crashing on any of them.
- Commit:
  ```bash
  git add . && git commit -m "module-07: RoomEngine end-to-end"
  ```

### Common pitfalls

1. **The engine crashes on an LLM timeout because you forgot to wrap
   the call.** Every `await self._llm....` call in the engine
   (transitively, including the ones inside agents) is wrapped in
   try/except that converts the failure to an error `Turn`. The
   first time you forget, the user sees a 500. Don't forget.

2. **The example index is empty because you only loaded examples,
   you didn't `await` the indexer.** Module 4's `ExampleIndexer.index`
   is async; calling it without `await` is a no-op. Symptom:
   retrieved_examples is always empty even after running
   `load_room.py`.

3. **You return a Turn with zero of the three fields set.** Module
   1's `__post_init__` raises. This catches the "I forgot to set
   `unable_to_answer` in this branch" bug at construction. If you
   see that error fire in real use, the engine has a logic hole —
   trace which branch produced the Turn.

4. **Conversation history isn't passed.** This module skips it. When
   you ask a follow-up question, the engine doesn't see prior turns.
   That's expected for the lab; Module 8 wires history through the
   API. Don't worry about it yet.

5. **The synthesis is mechanical and doesn't read well.** Yes. The
   stand-in is a placeholder; Module 9 builds a real
   `SynthesisAgent` that produces fluent prose. For now you're
   verifying the pipeline runs, not the synthesis quality.

6. **`load_room.py` fails the second time you run it because the
   examples are duplicated in Chroma.** Two options: (a) clear the
   collection before re-indexing (`vec._coll.delete()`); (b) use
   stable IDs so re-indexing is an upsert, not an append. The latter
   is what production code should do; for the lab, deleting and
   re-indexing is fine.

## Stretch

1. **Add conversation history.** Persist each `Turn` in
   `FileStoreProvider` keyed by `(conversation_id, turn_id)`. Load
   the most recent N turns on each `chat()` call and pass them
   through to the `ContextBuilder`. Test that a second question
   referencing "the previous result" gets context from the first
   turn.

2. **Add a `SynthesisAgent`.** It takes the question, the SQL, and
   the query result, and produces a plain-English summary with the
   causal-language regex scan from Module 6. Replace the
   `_synthesize_one_line` stand-in. Add a test that asserts the
   regex scan rejects a synthesis containing "caused by" and
   triggers a re-prompt.

3. **Add a streaming variant.** `RoomEngine.stream_chat()` that
   yields intermediate events: `status`, `sql`, `result`, `synth`,
   `done`. Same pipeline; different return type. This is what
   Module 8's API surface will consume.

## Reflection

1. The engine has six numbered steps and six try/except blocks. That
   looks repetitive. Would you DRY it up — say, a decorator that
   converts any exception in a method to an error Turn? What's the
   trade-off? (Hint: the explicit try/except blocks make the
   *specific* failure mode at each step easy to read. A decorator
   abstracts that.)

2. The `_default_caveats` helper makes the room's refusal list
   visible in every answer. Is that the right default, or should
   the agent only mention refusals when they're relevant to the
   specific question? What would "relevant" mean operationally?

3. Tiri's `RoomEngine` is ~900 lines; yours is ~150. What's in the
   extra 750? Skim Tiri's file and pick three sections that handle
   something yours doesn't. For each, decide: would your agent need
   this, and at what scale?

4. The pipeline currently has fixed shape (intent → SQL → execute →
   synth). What if the user asks a question that needs *two* SQL
   queries to answer (a year-over-year comparison, say)? Sketch how
   you'd extend the engine. Tiri's answer is `PlanningAgent` (EXT-1,
   in `docs/extensions.md`) — read that doc and compare to your
   sketch.
