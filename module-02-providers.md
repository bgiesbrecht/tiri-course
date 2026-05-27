# Module 2 — Providers as the I/O boundary

> **The rule:** Engine has zero I/O. Anything that talks to the
> outside world goes through a provider. No SDK imports in agents
> or orchestration.

This is the single most architectural decision in Tiri, and the one
most "build an agent in a weekend" tutorials get wrong. By the end of
this module, you'll have built two providers, written tests that don't
touch the network, and understood why that combination is what makes
the system survive contact with reality.

## Read

1. **`tiri/docs/providers.md`** — the whole thing. The interface
   contracts in this doc are the *only* place certain guarantees are
   specified. If you implement a provider that violates them, nothing
   else in the code will notice until production.

2. **`tiri/tiri/providers/base.py`** — read every abstract class. Pay
   attention to:
   - The class-level docstring "Contract (MUST):" sections. Those are
     the things that MUST be true of any implementation.
   - The `LLMProvider.complete` signature. The `task` and `model`
     parameters look optional but are how multi-model routing
     (Module 10) works. Single-backend implementations MUST accept
     them and MAY ignore them. Building this in from day one is
     cheaper than retrofitting later.
   - `MetadataProvider.enrich` and the "extends list fields — never
     replaces them" rule. Subtle. We'll come back to it in Module 4.

3. **Pick one concrete implementation to read end-to-end.**
   - If you have Databricks: `tiri/tiri/providers/databricks/llm.py`.
   - If not: `tiri/tiri/providers/local/llm_openai.py`.
   Read it cold, then re-read with the abstract `LLMProvider` open in
   another window. See how the implementation fits the contract.

## Concepts

### The boundary the engine never crosses

Open any file under `tiri/tiri/engine/` and search for `import
requests` or `import openai` or `import databricks`. You won't find
them. That is not coincidence; it is a design rule enforced by review.

The engine imports `LLMProvider` from `providers.base` and calls
`await llm.complete(...)`. It does not know — and is forbidden from
knowing — whether that call goes to OpenAI, Anthropic, Ollama,
Databricks Model Serving, or a fixture in a unit test.

This sounds like architectural overhead. It is actually what makes
six things possible:

1. **Unit tests run in 3 seconds.** No network.
2. **Same code works locally and in production.** Different container
   wiring, identical engine.
3. **Multi-model routing (Module 10) is a wiring change, not a
   refactor.** Replace `OpenAILLMProvider` with `RouterLLMProvider`
   and the engine doesn't notice.
4. **Per-user credentials (Module 7+) thread through the provider,
   not the engine.** The engine never sees a token.
5. **Adding a new backend is one file, not a rewrite.**
6. **Bugs surface at the boundary, not deep in the pipeline.** When
   `validate()` rejects a malformed SQL string before `execute()` runs,
   you debug a 50-line validator, not a five-agent pipeline.

### The seven providers

You will not implement all seven in this module — you'll do two
(`LLMProvider` and `QueryProvider`) and know what the others are for.
Each one wraps a different category of "thing the agent needs to talk
to":

- **`LLMProvider`** — calls a language model. Three operations:
  `complete()` (send messages, get a response), `stream()` (same but
  token-by-token for live UIs), `embed()` (turn text into vectors —
  see the concept box below). One interface, many backends: OpenAI,
  Anthropic, Ollama, Databricks Model Serving.

- **`CatalogProvider`** — answers "what tables exist, what columns,
  what types?" Reads the *physical* schema. In production, that's
  Unity Catalog; locally, that's DuckDB introspecting your Parquet
  files.

- **`MetadataProvider`** — answers "what do these tables and columns
  *mean*?" Descriptions, synonyms, business definitions. Multiple
  providers can stack (UC comments + a YAML file + room-specific
  overrides); Module 4 covers how they merge.

- **`QueryProvider`** — runs SQL against your data. Two operations:
  `validate()` (does this SQL parse and reference real tables?) and
  `execute()` (run it for real). Validation is mandatory before
  execution — see the concept box on why.

- **`VectorProvider`** — stores and searches *embeddings* (vector
  representations of text). Module 4 explains what these are.
  Locally: Chroma. In production: Databricks Vector Search.

- **`StoreProvider`** — persistence for rooms, conversation history,
  feedback. Like a tiny key-value store. Local: SQLite. Production:
  a Databricks table.

- **`MCPProvider`** — calls external tools via the Model Context
  Protocol. The extension point for "my data isn't all SQL — I have
  a documentation API, a ticketing system, an internal search."
  Module 10 covers this.

> **Concept: Embedding.** A way to turn text into a fixed-length
> list of numbers (a "vector"), such that two pieces of text with
> similar *meaning* produce vectors that are close together. "How
> much did we sell?" and "What was our revenue?" embed to nearby
> vectors even though they share no words. The LLM provider does the
> embedding for you — you just call `llm.embed(["text", "text"])`
> and get back a list of vectors.
>
> **Concept: Unity Catalog (UC).** Databricks' metadata layer:
> catalog → schema → table, with permissions, lineage, and
> table/column comments. Mentioned throughout as the production
> backing for the `CatalogProvider`. Locally you use a DuckDB-backed
> equivalent — same interface, simpler implementation.
>
> **Concept: MCP (Model Context Protocol).** An open standard for
> exposing tools to AI systems over a uniform interface. Think "REST,
> but designed for an agent on the other end." Tiri can act as an
> MCP server (other agents call Tiri) or an MCP client (Tiri calls
> external tools). Not relevant until Module 10.

### `validate()` before `execute()`

This is one of the most important patterns in the course. Most
tutorials show an agent that goes straight from "LLM generated SQL"
to "run the SQL." That is how you ship security holes and runaway
queries.

The pattern is: any SQL the LLM produces is *parsed and checked
against the real schema* before it touches a warehouse. If validation
fails, the agent either retries with the validator's error message
in its prompt (giving the model a chance to fix itself) or gives up
and reports an honest "I couldn't generate valid SQL for this
question." Either way, the warehouse never sees something broken or
malicious.

The contract on `QueryProvider`:

```python
async def validate(self, sql: str) -> ValidationResult: ...
async def execute(self, sql: str, ...) -> QueryResult: ...
```

Every caller MUST call `validate()` first. No exceptions. The
`RoomEngine` enforces this; if you write code that bypasses it, you
have introduced an SQL injection or a runaway query.

The validator is not just a parser. In production it is the
Databricks `EXPLAIN` endpoint, which catches:

- Syntax errors before they hit a warehouse.
- References to tables the caller can't see.
- Type mismatches that would crash mid-execution.

If `validate()` passes and `execute()` then fails, that is a bug worth
investigating — not "expected behavior."

### Contracts you can actually test

Abstract base classes in Python are easy to use sloppily — you can
implement the methods without honoring the contracts in their
docstrings. Tiri's discipline: every "MUST" in a contract is a test.

For example, `LLMProvider.complete` MUST "return deterministically at
temperature=0.0 for the same input." The test:

```python
async def test_local_llm_complete_is_deterministic(local_llm_provider):
    msgs = [LLMMessage(role="user", content="say hello")]
    r1 = await local_llm_provider.complete(msgs, temperature=0.0)
    r2 = await local_llm_provider.complete(msgs, temperature=0.0)
    assert r1.content == r2.content
```

This test will fail against a real LLM API for boring reasons (rate
limits, transient differences) — which is why your tests run against
a mock transport, not a real endpoint. The contract still applies.
The test catches the failure mode where a future maintainer
"helpfully" adds a default `temperature=0.7`.

## Lab

You'll build two providers (`LLMProvider`, `QueryProvider`) and their
tests. Seven checkpoints. Don't skip ahead.

### Checkpoint 1 — The abstract base classes

In `~/my-agent/`:

```bash
mkdir -p agent/providers
touch agent/providers/__init__.py agent/providers/base.py
```

Also add the result dataclasses to `agent/data_models.py`. Append:

```python
@dataclass
class LLMMessage:
    role: Literal["system", "user", "assistant"]
    content: str

    def __post_init__(self) -> None:
        if self.role not in ("system", "user", "assistant"):
            raise ValueError(f"bad role: {self.role!r}")


@dataclass
class LLMResponse:
    content: str
    model: str = ""        # the model that produced this response
    usage: dict = field(default_factory=dict)   # tokens, etc. — optional


@dataclass
class ValidationResult:
    is_valid: bool
    error: str | None = None


@dataclass
class QueryResult:
    columns: list[str]
    rows: list[list]
    row_count: int
```

Now `agent/providers/base.py`:

```python
"""Abstract provider interfaces. Every external I/O goes through one."""
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import AsyncIterator

from agent.data_models import LLMMessage, LLMResponse, QueryResult, ValidationResult


# ─── Errors ─────────────────────────────────────────────────────────

class ProviderError(Exception):
    """Base class for all provider failures."""


class LLMProviderError(ProviderError):
    """Raised by LLMProvider implementations on any failure mode."""


class QueryProviderError(ProviderError):
    """Raised by QueryProvider implementations on any failure mode."""


# ─── LLMProvider ────────────────────────────────────────────────────

class LLMProvider(ABC):
    """Abstracts language-model calls: completion, streaming, embedding.

    Contract (MUST):
      - complete() returns deterministically at temperature=0.0 for
        the same input.
      - stream() yields the same total content as complete() would
        for the same input.
      - embed() returns one vector per input text, in input order.
      - All methods raise LLMProviderError on failure, never raw SDK
        exceptions.
    """

    @abstractmethod
    async def complete(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        max_tokens: int = 2048,
        task: str = "default",      # routing hint for Module 10; ignore for now
        model: str | None = None,   # per-call model override (Module 10)
    ) -> LLMResponse:
        ...

    @abstractmethod
    async def stream(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        task: str = "default",
        model: str | None = None,
    ) -> AsyncIterator[str]:
        ...

    @abstractmethod
    async def embed(self, texts: list[str]) -> list[list[float]]:
        ...


# ─── QueryProvider ──────────────────────────────────────────────────

class QueryProvider(ABC):
    """Abstracts SQL validation and execution.

    Contract (MUST):
      - validate() never raises for SQL errors — it returns a
        ValidationResult with is_valid=False. It MAY raise for
        infrastructure errors (connection down, etc.).
      - execute() runs the SQL and returns rows; it MAY raise.
      - Any caller MUST call validate() before execute(). The engine
        enforces this; providers don't have to re-check.
    """

    @abstractmethod
    async def validate(self, sql: str) -> ValidationResult:
        ...

    @abstractmethod
    async def execute(self, sql: str, row_limit: int = 10_000) -> QueryResult:
        ...
```

The `task` and `model` parameters on `LLMProvider` look like dead
code right now — they are. Module 10 turns them on (multi-model
routing). Adding them on day one is ~5 lines; retrofitting later is
a refactor across every consumer. Pay the small cost now.

### Checkpoint 2 — Implement `LLMProvider`

Pick one backend — whichever key you have. The course will use
OpenAI in examples; the Anthropic path is the stretch.

Install:

```bash
pip install openai
```

Create `agent/providers/openai_llm.py`:

```python
"""OpenAI-backed LLMProvider."""
from __future__ import annotations
from typing import AsyncIterator

from openai import AsyncOpenAI, OpenAIError

from agent.data_models import LLMMessage, LLMResponse
from agent.providers.base import LLMProvider, LLMProviderError


class OpenAILLMProvider(LLMProvider):
    def __init__(
        self,
        api_key: str,
        model: str = "gpt-4o-mini",
        embed_model: str = "text-embedding-3-small",
    ) -> None:
        self._client = AsyncOpenAI(api_key=api_key)
        self._model = model
        self._embed_model = embed_model

    async def complete(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        max_tokens: int = 2048,
        task: str = "default",
        model: str | None = None,
    ) -> LLMResponse:
        try:
            resp = await self._client.chat.completions.create(
                model=model or self._model,
                messages=[{"role": m.role, "content": m.content} for m in messages],
                temperature=temperature,
                max_tokens=max_tokens,
            )
        except OpenAIError as e:
            raise LLMProviderError(f"OpenAI complete() failed: {e}") from e
        content = resp.choices[0].message.content or ""
        return LLMResponse(content=content, model=resp.model)

    async def stream(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        task: str = "default",
        model: str | None = None,
    ) -> AsyncIterator[str]:
        # TODO: implement streaming. Hint: client.chat.completions.create(stream=True)
        # returns an async iterator of chunks; each chunk has .choices[0].delta.content.
        raise NotImplementedError("Stretch goal — required by Module 8, not now.")

    async def embed(self, texts: list[str]) -> list[list[float]]:
        try:
            resp = await self._client.embeddings.create(
                model=self._embed_model,
                input=texts,
            )
        except OpenAIError as e:
            raise LLMProviderError(f"OpenAI embed() failed: {e}") from e
        return [item.embedding for item in resp.data]
```

Notice: every `OpenAIError` is caught and re-raised as
`LLMProviderError`. Why? Because consumers two layers up should not
need to know your provider is OpenAI to handle errors. They catch
`LLMProviderError`. If you later swap to Anthropic, the catch sites
don't change.

### Checkpoint 3 — Test `LLMProvider` without the network

We can't hit the real OpenAI API in tests — too slow, not
deterministic, costs money. Mock the HTTP layer.

Install:

```bash
pip install pytest-asyncio respx httpx
```

Add to `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

Create `tests/test_llm_provider.py`:

```python
import pytest
import respx
import httpx
from agent.data_models import LLMMessage
from agent.providers.openai_llm import OpenAILLMProvider
from agent.providers.base import LLMProviderError


@pytest.mark.asyncio
@respx.mock
async def test_complete_returns_mock_content():
    respx.post("https://api.openai.com/v1/chat/completions").mock(
        return_value=httpx.Response(
            200,
            json={
                "id": "x", "object": "chat.completion", "created": 0,
                "model": "gpt-4o-mini",
                "choices": [
                    {
                        "index": 0,
                        "message": {"role": "assistant", "content": "Hello, world."},
                        "finish_reason": "stop",
                    }
                ],
                "usage": {"prompt_tokens": 5, "completion_tokens": 3, "total_tokens": 8},
            },
        )
    )

    llm = OpenAILLMProvider(api_key="sk-test")
    resp = await llm.complete([LLMMessage(role="user", content="hi")])
    assert resp.content == "Hello, world."


@pytest.mark.asyncio
@respx.mock
async def test_complete_wraps_openai_errors():
    respx.post("https://api.openai.com/v1/chat/completions").mock(
        return_value=httpx.Response(500, json={"error": {"message": "boom"}})
    )

    llm = OpenAILLMProvider(api_key="sk-test")
    with pytest.raises(LLMProviderError, match="complete"):
        await llm.complete([LLMMessage(role="user", content="hi")])


@pytest.mark.asyncio
@respx.mock
async def test_embed_returns_vectors_in_order():
    respx.post("https://api.openai.com/v1/embeddings").mock(
        return_value=httpx.Response(
            200,
            json={
                "object": "list",
                "data": [
                    {"index": 0, "object": "embedding", "embedding": [0.1, 0.2]},
                    {"index": 1, "object": "embedding", "embedding": [0.3, 0.4]},
                ],
                "model": "text-embedding-3-small",
                "usage": {"prompt_tokens": 2, "total_tokens": 2},
            },
        )
    )

    llm = OpenAILLMProvider(api_key="sk-test")
    vecs = await llm.embed(["foo", "bar"])
    assert vecs == [[0.1, 0.2], [0.3, 0.4]]
```

Run:

```bash
pytest tests/test_llm_provider.py -v
```

Expected output:

```
tests/test_llm_provider.py::test_complete_returns_mock_content PASSED
tests/test_llm_provider.py::test_complete_wraps_openai_errors PASSED
tests/test_llm_provider.py::test_embed_returns_vectors_in_order PASSED
```

If the second test fails with `OpenAIError` instead of
`LLMProviderError`, your `except` clause isn't catching everything
— check that you wrapped both the `complete` and `embed` calls.

### Checkpoint 4 — Fixture data for `QueryProvider`

You need a tiny test database. Pick a small domain (mirror Module 0's
"pick your domain" — could be the same one).

Install:

```bash
pip install duckdb
```

Create `tests/conftest.py`:

```python
import pytest
import duckdb
from pathlib import Path


@pytest.fixture
def fixture_db(tmp_path: Path) -> Path:
    """One-table DuckDB fixture: data/orders.parquet."""
    data_dir = tmp_path / "data"
    data_dir.mkdir()
    con = duckdb.connect(":memory:")
    con.execute("""
        CREATE TABLE orders (
            order_id INTEGER,
            customer_id INTEGER,
            amount DECIMAL(10, 2),
            region VARCHAR
        )
    """)
    con.execute("""
        INSERT INTO orders VALUES
            (1, 100, 49.99, 'NORTH'),
            (2, 101, 199.00, 'SOUTH'),
            (3, 100, 12.50, 'NORTH'),
            (4, 102, 75.25, 'EAST')
    """)
    con.execute(
        f"COPY orders TO '{data_dir}/orders.parquet' (FORMAT PARQUET)"
    )
    return data_dir
```

### Checkpoint 5 — Implement `QueryProvider` over DuckDB

Create `agent/providers/duckdb_query.py`:

```python
"""DuckDB-backed QueryProvider — reads Parquet files in a directory."""
from __future__ import annotations
from pathlib import Path

import duckdb

from agent.data_models import QueryResult, ValidationResult
from agent.providers.base import QueryProvider, QueryProviderError


class DuckDBQueryProvider(QueryProvider):
    def __init__(self, data_dir: str) -> None:
        self._data_dir = Path(data_dir)

    def _connect(self) -> duckdb.DuckDBPyConnection:
        """Fresh connection + register every parquet file as a table."""
        con = duckdb.connect(":memory:")
        for parquet in self._data_dir.glob("*.parquet"):
            con.execute(
                f"CREATE VIEW {parquet.stem} AS "
                f"SELECT * FROM read_parquet('{parquet}')"
            )
        return con

    async def validate(self, sql: str) -> ValidationResult:
        con = self._connect()
        try:
            con.execute(f"EXPLAIN {sql}")
        except duckdb.Error as e:
            return ValidationResult(is_valid=False, error=str(e))
        return ValidationResult(is_valid=True)

    async def execute(
        self, sql: str, row_limit: int = 10_000
    ) -> QueryResult:
        con = self._connect()
        try:
            cur = con.execute(f"SELECT * FROM ({sql}) LIMIT {row_limit}")
        except duckdb.Error as e:
            raise QueryProviderError(f"execute() failed: {e}") from e
        cols = [d[0] for d in cur.description]
        rows = [list(r) for r in cur.fetchall()]
        return QueryResult(columns=cols, rows=rows, row_count=len(rows))
```

Two things to notice:

1. **`validate()` calls DuckDB's `EXPLAIN`.** DuckDB parses and
   plans the query without running it. Syntax errors and unknown
   tables are caught. Cost is microseconds. This is the real-world
   pattern: validators don't have to be hand-written; the database
   engine usually has one for free.

2. **`validate()` doesn't raise on SQL errors.** It returns
   `is_valid=False`. The engine layer (Module 7) wants to use that
   structured result to retry the SQL agent with the error message
   in its prompt. If `validate()` raised, the engine would have to
   try/except, which is awkward and easy to miss.

### Checkpoint 6 — Test `QueryProvider`

`tests/test_query_provider.py`:

```python
import pytest
from agent.providers.duckdb_query import DuckDBQueryProvider
from agent.providers.base import QueryProviderError


@pytest.mark.asyncio
async def test_validate_accepts_correct_sql(fixture_db):
    q = DuckDBQueryProvider(data_dir=str(fixture_db))
    result = await q.validate("SELECT count(*) FROM orders")
    assert result.is_valid is True
    assert result.error is None


@pytest.mark.asyncio
async def test_validate_rejects_unknown_table(fixture_db):
    q = DuckDBQueryProvider(data_dir=str(fixture_db))
    result = await q.validate("SELECT * FROM nonexistent_table")
    assert result.is_valid is False
    assert "nonexistent_table" in result.error.lower() or \
           "does not exist" in result.error.lower() or \
           "catalog" in result.error.lower()


@pytest.mark.asyncio
async def test_validate_rejects_syntax_error(fixture_db):
    q = DuckDBQueryProvider(data_dir=str(fixture_db))
    result = await q.validate("SELECT FROM WHERE")
    assert result.is_valid is False


@pytest.mark.asyncio
async def test_execute_returns_correct_count(fixture_db):
    q = DuckDBQueryProvider(data_dir=str(fixture_db))
    result = await q.execute("SELECT region, COUNT(*) AS n FROM orders GROUP BY region")
    by_region = {row[0]: row[1] for row in result.rows}
    assert by_region["NORTH"] == 2
    assert by_region["SOUTH"] == 1
    assert by_region["EAST"] == 1


@pytest.mark.asyncio
async def test_execute_raises_on_bad_sql(fixture_db):
    q = DuckDBQueryProvider(data_dir=str(fixture_db))
    with pytest.raises(QueryProviderError):
        await q.execute("SELECT * FROM nonexistent_table")
```

Run:

```bash
pytest tests/ -v
```

Expected output (all tests from Modules 1 and 2 combined):

```
tests/test_data_models.py::test_question_constructs PASSED
... (other Module 1 tests) ...
tests/test_llm_provider.py::test_complete_returns_mock_content PASSED
tests/test_llm_provider.py::test_complete_wraps_openai_errors PASSED
tests/test_llm_provider.py::test_embed_returns_vectors_in_order PASSED
tests/test_query_provider.py::test_validate_accepts_correct_sql PASSED
tests/test_query_provider.py::test_validate_rejects_unknown_table PASSED
tests/test_query_provider.py::test_validate_rejects_syntax_error PASSED
tests/test_query_provider.py::test_execute_returns_correct_count PASSED
tests/test_query_provider.py::test_execute_raises_on_bad_sql PASSED

========== 15 passed in 0.20s ==========
```

### Checkpoint 7 — The boundary check (verify zero engine I/O)

You don't have an engine yet, but the discipline starts now. Run:

```bash
grep -rn "import openai\|import anthropic\|import duckdb\|import httpx" agent/
```

Expected output — these imports appear ONLY in `agent/providers/`:

```
agent/providers/openai_llm.py:6:from openai import AsyncOpenAI, OpenAIError
agent/providers/duckdb_query.py:6:import duckdb
```

If you see any of those imports in `agent/data_models.py` or anywhere
else outside `providers/`, something has gone wrong — fix it before
moving on. The whole architecture depends on this property staying
true.

### Definition of done

- `agent/providers/base.py` defines `LLMProvider` and `QueryProvider`
  as abstract base classes, with the contracts in their docstrings.
- `agent/providers/openai_llm.py` (or `anthropic_llm.py`) implements
  `LLMProvider`. Errors are wrapped in `LLMProviderError`.
- `agent/providers/duckdb_query.py` implements `QueryProvider`. The
  `validate()` method does not raise on SQL errors; the `execute()`
  method does.
- All 15+ tests pass.
- `grep` confirms no SDK imports outside `agent/providers/`.
- Commit:
  ```bash
  git add . && git commit -m "module-02: LLMProvider and QueryProvider with mocked tests"
  ```

### Common pitfalls

1. **`pytest` runs sync tests but skips async ones.** Symptom: your
   tests "pass" but only because nothing inside them actually ran.
   Fix: install `pytest-asyncio` and add `asyncio_mode = "auto"` to
   `pyproject.toml`. Or use `@pytest.mark.asyncio` on every async
   test.

2. **`respx` mocks the wrong URL.** OpenAI's base URL is
   `https://api.openai.com/v1/`. If you use `OPENAI_BASE_URL` to
   point at a local proxy, your mock won't match. Stick with the
   default for the test.

3. **You catch broad `Exception` in `validate()`.** Tempting. Don't.
   Catch `duckdb.Error` (or `OpenAIError`) specifically. Broad
   catches will silently swallow real bugs (typos in your provider
   code) and report them as "invalid SQL."

4. **You forget that `EXPLAIN` requires DuckDB to know the table.**
   `EXPLAIN SELECT * FROM orders` only works if `orders` is
   registered in the connection. Our `_connect()` registers every
   `.parquet` file as a view. If you create one connection and reuse
   it across tests, you may have stale state — that's why
   `_connect()` makes a fresh one each call.

5. **You wrap the wrong exception layer in `LLMProviderError`.** The
   `openai` SDK's exception hierarchy has `OpenAIError` at the top,
   with `APIError`, `RateLimitError`, etc. underneath. Catch the
   top-level class so subclass exceptions don't slip past your wrap.

## Stretch

Implement a third provider — a `VectorProvider` over Chroma — and
write a test that:

1. Indexes three example texts with their embeddings.
2. Queries for the nearest neighbor of a fourth text.
3. Asserts the returned text is the expected one.

If you're feeling ambitious, swap your `LLMProvider`'s embed
implementation for a deterministic fake (e.g., a hash-based vector)
and rerun the test. The Chroma index should still return the right
neighbor, because semantic correctness lives in the embeddings, not
the index. Notice how cleanly this separation lets you test the
*index* in isolation from the *model*.

## Reflection

1. The contract on `LLMProvider.complete` says it MUST raise
   `LLMProviderError` on failure, not the SDK's native exception
   class. Why does that matter? What goes wrong if you let the raw
   SDK exception propagate? (Hint: think about who catches it three
   layers up, and what they know about your provider choice.)

2. Your `QueryProvider` has separate `validate()` and `execute()`
   methods. Why not collapse them into one method that validates
   internally before executing? What capability would you lose? (Hint:
   think about a UI that wants to highlight invalid SQL before the
   user clicks "run", or a test that asserts SQL is *generated*
   correctly without actually running it.)

3. You implemented `LLMProvider` and `QueryProvider` independently.
   Sketch the function signature of the function that would *use*
   both — e.g., "generate SQL from a question and run it." Where
   does that function live? Importantly: does it live in your
   `providers/` module, or somewhere else? Why?
