# Module 4 — Knowledge: metadata + retrieval

> **The rule:** Context is computed fresh per request. Caching room
> config or schemas across requests is how you ship a stale-prompt
> bug.

In this module you'll build the layer that turns a raw question into
*the bundle of facts an agent needs to answer it*. Tiri calls this
bundle the `ContextPackage`. Building it well is most of what makes
the difference between an agent that gets the right SQL and an agent
that confabulates.

Two operating modes for this module's lab:

- **Local mode (no Databricks):** DuckDB + a fixture Parquet file +
  Chroma in-memory. This is what the module body walks through.
- **Databricks mode:** Unity Catalog + Databricks Vector Search.
  Same code shape, different `_build_*` calls in your container. The
  module flags the differences as they come up.

## Read

1. **`tiri/docs/knowledge_store.md`** — the whole thing. Pay
   attention to:
   - The `ContextPackage` fields and what populates each.
   - The "metadata stack" idea: multiple providers contribute, in
     priority order, with explicit conflict tracking.
   - The "indexed examples" pattern: 5–20 example questions per room,
     embedded, retrieved top-k at request time.

2. **`tiri/docs/metadata.md`** — the whole thing. The merge rules in
   particular: list fields *extend*, scalar fields *override* (with
   a `MetadataConflict` recorded). This is subtle and worth a slow
   read.

3. **`tiri/tiri/knowledge/context_builder.py`** — read end-to-end.
   It's short. Notice:
   - Zero `llm.complete()` calls. The only LLM call in this layer is
     `llm.embed()` inside `ExampleIndexer.retrieve`. This is a data
     assembly layer, not a reasoning layer.
   - The order of operations: select tables → fetch metadata →
     retrieve examples → assemble `ContextPackage`. Each step's
     output feeds the next.

4. **`tiri/tiri/knowledge/metadata_fetcher.py`** — skim. The class
   coordinates the catalog (physical schema) and the metadata
   providers (semantic enrichment). Focus on the order: catalog
   *first*, then providers in priority order.

5. **`tiri/tiri/knowledge/example_indexer.py`** — skim.

6. **`tiri/demo/tpch_metadata.yaml`** — read in full. This is what a
   real metadata YAML looks like. Notice schema-level fields, table
   descriptions, synonyms, grain, semantic types per column.

## Concepts

### Why a `ContextPackage`

Before any agent generates anything, it needs a closed-form bundle of
facts: which tables, which columns, which examples, which joins,
which mandatory filters, which conversation history. That bundle is
the `ContextPackage`.

Making it a single object is not a stylistic choice. Three reasons:

1. **Reproducibility.** Save the `ContextPackage` next to the answer
   and you can replay why the agent answered what it answered, weeks
   later, without rebuilding the catalog state.

2. **Testability.** Every agent in Module 6 will take a
   `ContextPackage` as input. You can construct a fixture
   `ContextPackage` and assert what the agent does with it — no
   provider mocking required at the agent layer.

3. **Boundary clarity.** Anything not in the `ContextPackage` is
   *not* available to the agent. If an agent needs a new fact, you
   add it to `ContextPackage` and to `ContextBuilder`. There's no
   side channel.

### The metadata stack

A table like `samples.tpch.lineitem` has facts from multiple sources:

- **Physical schema** (Unity Catalog or DuckDB introspection): column
  names, data types, row count. Authoritative for what *exists*.

- **Catalog annotations** (UC table/column comments, or your YAML
  file): descriptions, synonyms, business meaning. Authoritative for
  what things *mean* in this organization.

- **Room config overrides** (the room author's per-column overrides):
  "in this room, treat `o_orderstatus = 'F'` as fulfilled orders." 
  Highest priority — overrides the YAML which overrides the catalog.

Tiri stacks these providers in priority order. The contract:

- **Catalog goes first.** It owns physical facts (column names, data
  types). Nobody else may change them.
- **Metadata providers run after, in declared order.** Each MAY add
  to list fields (synonyms, sample values, etc.) and MAY override
  scalar fields (descriptions, grain). When a scalar override
  happens, a `MetadataConflict` is recorded with the previous value
  and the source that overwrote it.

The conflict record matters. It means a room maintainer can ask "why
does the agent think `customer.c_mktsegment` means 'industry vertical'
when the UC comment says something else?" and get a precise answer:
"because `tpch_metadata.yaml` overrode it on line 47, here's the
prior value."

### Retrieval over stuffing

A naive agent does `SELECT * FROM information_schema` and crams every
table into the prompt. This:

- Costs a lot of tokens.
- Confuses the model with irrelevant tables.
- Gets worse, not better, as your schema grows.

Tiri's approach: index 5–20 example Q&A pairs per room, embed them at
indexing time, and at request time embed the new question and pull
the top-k nearest examples. The retrieved examples come with the
*correct SQL*, the *correct tables*, and *natural-language phrasing
typical of this room*. They tell the LLM, by example, what kind of
question this is and how to answer it.

This is RAG, but disciplined: small example sets that the room
author curates, not "let me embed the entire data warehouse."

### Fresh per request

The same room config is used for two different questions five minutes
apart. Why rebuild the `ContextPackage` for each one?

Because *the question changes which tables and examples are
relevant*. The top-k examples for "revenue by region" are different
from the top-k for "average shipping delay." The selected wildcards
(Module 10) expand differently. The retrieved metadata excludes
tables the question doesn't reference.

Reusing a `ContextPackage` across requests is a category error, like
caching the result of a database query that has parameters. Don't.

The thing you *can* cache, judiciously, is the *catalog read*. Unity
Catalog table schemas don't change every 30 seconds. Tiri exposes
`metadata_cache_ttl` for this. The `ContextPackage` itself is always
fresh; the underlying physical schema fetch may come from a TTL
cache.

## Lab

You'll build a slimmed-down `ContextBuilder` for your agent. It will:

1. Read physical schema from DuckDB.
2. Enrich with a small YAML metadata file you write.
3. Retrieve top-k examples from a Chroma collection.
4. Return a `ContextPackage`.

### Prep — fixture data

Pick a tiny domain you understand. The lab assumes one of these
shapes; adapt as needed:

- **Sales:** `orders` (order_id, customer_id, amount, region, ts)
- **Library:** `books` (book_id, title, author, genre, year)
- **Issues:** `tickets` (ticket_id, status, priority, opened_at)

Pick one. Generate ~1000 rows of synthetic data. Save as
`~/my-agent/data/<table>.parquet`. (DuckDB can read CSV too — Parquet
is just compact.)

Verify it loads:

```bash
python -c "
import duckdb
con = duckdb.connect(':memory:')
con.execute(\"SELECT * FROM read_parquet('data/orders.parquet') LIMIT 5\").fetchall()
print('ok')
"
```

### Checkpoint 1 — Metadata YAML

Create `~/my-agent/agent_metadata.yaml`:

```yaml
schemas:
  main:
    description: |
      The default schema. Contains transactional data from the
      production order system.
    date_range: "2024-01-01 to present"

tables:
  orders:
    description: One row per customer order.
    synonyms: [sales, purchases]
    grain: order
    columns:
      amount:
        description: Order total in USD.
        semantic_type: currency
        sample_values: [12.99, 49.50, 199.00]
      region:
        description: Customer's home region.
        semantic_type: dimension
        sample_values: [NORTH, SOUTH, EAST, WEST]
```

This is small on purpose. Real rooms have richer YAML; you can grow
it later. What matters now is that the agent will see "amount means
currency in USD" rather than just "amount, float".

### Checkpoint 2 — Extend the data models

In `agent/data_models.py`, add:

```python
@dataclass
class ColumnMeta:
    name: str
    data_type: str
    description: str = ""
    synonyms: list[str] = field(default_factory=list)
    sample_values: list[Any] = field(default_factory=list)
    semantic_type: str = ""


@dataclass
class TableMeta:
    full_name: str       # e.g. "main.orders"
    description: str = ""
    synonyms: list[str] = field(default_factory=list)
    grain: str = ""
    columns: list[ColumnMeta] = field(default_factory=list)
    metadata_sources: list[str] = field(default_factory=list)
    # Tracks which providers contributed. Future-you debugging will be grateful.


@dataclass
class ExampleSQL:
    question: str
    sql: str
    explanation: str = ""


@dataclass
class ContextPackage:
    question: str
    tables: dict[str, TableMeta]     # keyed by full_name
    retrieved_examples: list[ExampleSQL]
    schema_descriptions: dict[str, str]   # keyed by schema name
```

Note: this is a *slimmed* `ContextPackage` — no joins, no metrics, no
domain knowledge, no MCP. Add fields when the agent needs them. The
discipline is to start narrow and grow.

### Checkpoint 3 — Catalog provider over DuckDB

Add `agent/providers/duckdb_catalog.py`:

```python
"""Physical schema introspection over DuckDB."""
from __future__ import annotations
import duckdb
from agent.data_models import TableMeta, ColumnMeta
from agent.providers.base import CatalogProvider, CatalogProviderError


class DuckDBCatalogProvider(CatalogProvider):
    def __init__(self, data_dir: str) -> None:
        self._data_dir = data_dir

    async def get_table_meta(self, full_name: str) -> TableMeta:
        # full_name: "main.orders" → read data_dir/orders.parquet
        parts = full_name.split(".")
        if len(parts) != 2:
            raise CatalogProviderError(
                f"expected '<schema>.<table>', got {full_name!r}"
            )
        _, table = parts
        path = f"{self._data_dir}/{table}.parquet"
        con = duckdb.connect(":memory:")
        try:
            schema = con.execute(
                f"DESCRIBE SELECT * FROM read_parquet('{path}')"
            ).fetchall()
        except duckdb.IOException as e:
            raise CatalogProviderError(
                f"table {full_name!r} not found at {path}"
            ) from e

        cols = [
            ColumnMeta(name=row[0], data_type=row[1])
            for row in schema
        ]
        return TableMeta(
            full_name=full_name,
            columns=cols,
            metadata_sources=["duckdb_catalog"],
        )
```

You'll need to define the abstract `CatalogProvider` in
`agent/providers/base.py` if you haven't yet. Mirror the contract
from Tiri's `providers/base.py`.

Test:

```python
def test_duckdb_catalog_reads_schema(tmp_path):
    import asyncio
    import duckdb
    # Create a fixture parquet file
    con = duckdb.connect(":memory:")
    con.execute("CREATE TABLE orders (order_id INTEGER, amount DECIMAL, region VARCHAR)")
    con.execute(
        f"COPY orders TO '{tmp_path}/orders.parquet' (FORMAT PARQUET)"
    )

    from agent.providers.duckdb_catalog import DuckDBCatalogProvider
    cat = DuckDBCatalogProvider(data_dir=str(tmp_path))
    meta = asyncio.run(cat.get_table_meta("main.orders"))

    assert meta.full_name == "main.orders"
    assert {c.name for c in meta.columns} == {"order_id", "amount", "region"}
    assert "duckdb_catalog" in meta.metadata_sources
```

### Checkpoint 4 — YAML metadata provider

Add `agent/providers/yaml_metadata.py`:

```python
"""Reads agent_metadata.yaml and enriches TableMeta in place."""
from __future__ import annotations
import yaml
from pathlib import Path
from agent.data_models import TableMeta, ColumnMeta
from agent.providers.base import MetadataProvider


class YAMLMetadataProvider(MetadataProvider):
    def __init__(self, yaml_path: str) -> None:
        with Path(yaml_path).open() as f:
            self._raw = yaml.safe_load(f) or {}

    @property
    def name(self) -> str:
        return "yaml"

    async def enrich(self, tables: dict[str, TableMeta]) -> None:
        for full_name, meta in tables.items():
            table_key = full_name.split(".")[-1]
            spec = (self._raw.get("tables") or {}).get(table_key)
            if not spec:
                continue
            # Scalar override (description, grain) — only when current is empty.
            if not meta.description and spec.get("description"):
                meta.description = spec["description"]
            if not meta.grain and spec.get("grain"):
                meta.grain = spec["grain"]
            # List extension (synonyms) — always append, deduped.
            meta.synonyms = list({*meta.synonyms, *(spec.get("synonyms") or [])})
            # Per-column enrichment.
            col_specs = spec.get("columns") or {}
            for col in meta.columns:
                cspec = col_specs.get(col.name)
                if not cspec:
                    continue
                if not col.description and cspec.get("description"):
                    col.description = cspec["description"]
                if not col.semantic_type and cspec.get("semantic_type"):
                    col.semantic_type = cspec["semantic_type"]
                col.sample_values = list({
                    *col.sample_values, *(cspec.get("sample_values") or [])
                })
            meta.metadata_sources.append(self.name)
```

Test:

```python
def test_yaml_metadata_enriches_table(tmp_path):
    import asyncio
    yaml_path = tmp_path / "meta.yaml"
    yaml_path.write_text("""
tables:
  orders:
    description: Customer orders.
    synonyms: [sales]
    columns:
      amount:
        description: Amount in USD.
        semantic_type: currency
""")
    from agent.providers.yaml_metadata import YAMLMetadataProvider
    from agent.data_models import TableMeta, ColumnMeta
    prov = YAMLMetadataProvider(str(yaml_path))
    tables = {
        "main.orders": TableMeta(
            full_name="main.orders",
            columns=[ColumnMeta(name="amount", data_type="DECIMAL")],
            metadata_sources=["duckdb_catalog"],
        )
    }
    asyncio.run(prov.enrich(tables))

    t = tables["main.orders"]
    assert t.description == "Customer orders."
    assert "sales" in t.synonyms
    assert t.columns[0].description == "Amount in USD."
    assert t.columns[0].semantic_type == "currency"
    assert t.metadata_sources == ["duckdb_catalog", "yaml"]
```

### Checkpoint 5 — Example indexer over Chroma

Add `agent/providers/chroma_vector.py`:

```python
"""Vector index over Chroma. Embeddings come from the LLM provider."""
from __future__ import annotations
import chromadb
from agent.providers.base import VectorProvider, LLMProvider


class ChromaVectorProvider(VectorProvider):
    def __init__(self, path: str = ":memory:", collection: str = "examples") -> None:
        self._client = (
            chromadb.Client() if path == ":memory:"
            else chromadb.PersistentClient(path=path)
        )
        self._coll = self._client.get_or_create_collection(collection)

    async def upsert(
        self, ids: list[str], texts: list[str], embeddings: list[list[float]]
    ) -> None:
        self._coll.upsert(ids=ids, documents=texts, embeddings=embeddings)

    async def query(
        self, embedding: list[float], top_k: int = 3
    ) -> list[tuple[str, str, float]]:
        res = self._coll.query(query_embeddings=[embedding], n_results=top_k)
        ids = res["ids"][0]
        docs = res["documents"][0]
        dists = res["distances"][0]
        return list(zip(ids, docs, dists))
```

Then a thin `ExampleIndexer` in `agent/knowledge/example_indexer.py`:

```python
"""Indexes example Q&A pairs and retrieves top-k for a new question."""
from __future__ import annotations
import json
from agent.data_models import ExampleSQL
from agent.providers.base import LLMProvider, VectorProvider


class ExampleIndexer:
    def __init__(self, llm: LLMProvider, vector: VectorProvider) -> None:
        self._llm = llm
        self._vector = vector

    async def index(self, examples: list[ExampleSQL]) -> None:
        texts = [e.question for e in examples]
        embeddings = await self._llm.embed(texts)
        # JSON-encode the full example so we can round-trip it from the index.
        ids = [f"ex-{i}" for i in range(len(examples))]
        await self._vector.upsert(
            ids=ids,
            texts=[json.dumps(e.__dict__) for e in examples],
            embeddings=embeddings,
        )

    async def retrieve(self, question: str, top_k: int = 3) -> list[ExampleSQL]:
        [emb] = await self._llm.embed([question])
        hits = await self._vector.query(emb, top_k=top_k)
        return [ExampleSQL(**json.loads(doc)) for _, doc, _ in hits]
```

### Checkpoint 6 — The `ContextBuilder`

Finally, tie it together in `agent/knowledge/context_builder.py`:

```python
"""Assembles a ContextPackage per request."""
from __future__ import annotations
from agent.data_models import ContextPackage, ExampleSQL, RoomConfig  # ← see note
from agent.providers.base import (
    CatalogProvider, LLMProvider, MetadataProvider, VectorProvider
)
from agent.knowledge.example_indexer import ExampleIndexer


class ContextBuilder:
    def __init__(
        self,
        catalog: CatalogProvider,
        metadata_providers: list[MetadataProvider],
        llm: LLMProvider,
        vector: VectorProvider,
    ) -> None:
        self._catalog = catalog
        self._metadata_providers = metadata_providers
        self._llm = llm
        self._vector = vector

    async def build(
        self,
        question: str,
        tables: list[str],   # e.g. ["main.orders"]
        top_k_examples: int = 3,
    ) -> ContextPackage:
        # 1. Physical schema from the catalog.
        table_metas = {}
        for full_name in tables:
            table_metas[full_name] = await self._catalog.get_table_meta(full_name)

        # 2. Enrich with each metadata provider, in priority order.
        for prov in self._metadata_providers:
            await prov.enrich(table_metas)

        # 3. Retrieve top-k examples.
        indexer = ExampleIndexer(self._llm, self._vector)
        retrieved = await indexer.retrieve(question, top_k=top_k_examples)

        return ContextPackage(
            question=question,
            tables=table_metas,
            retrieved_examples=retrieved,
            schema_descriptions={},   # leave empty for now; lab stretch goal
        )
```

You'll need a minimal `RoomConfig` dataclass too, if you don't have
one — but for this lab, the list of tables is passed directly to
`build()`, so you can defer `RoomConfig` to Module 7. (Note the
import comment above.)

### Checkpoint 7 — The integration test

This is the one that proves the whole layer works.

```python
# tests/test_context_builder.py
import asyncio, json
import pytest
import duckdb

from agent.data_models import ExampleSQL
from agent.providers.duckdb_catalog import DuckDBCatalogProvider
from agent.providers.yaml_metadata import YAMLMetadataProvider
from agent.providers.chroma_vector import ChromaVectorProvider
from agent.knowledge.example_indexer import ExampleIndexer
from agent.knowledge.context_builder import ContextBuilder


@pytest.fixture
def fixture_data(tmp_path):
    con = duckdb.connect(":memory:")
    con.execute(
        "CREATE TABLE orders ("
        "  order_id INTEGER, customer_id INTEGER, amount DECIMAL, region VARCHAR"
        ")"
    )
    con.execute("INSERT INTO orders VALUES (1, 100, 49.99, 'NORTH')")
    con.execute(f"COPY orders TO '{tmp_path}/orders.parquet' (FORMAT PARQUET)")
    yaml_path = tmp_path / "meta.yaml"
    yaml_path.write_text("""
tables:
  orders:
    description: Customer orders.
    columns:
      amount:
        description: Order total in USD.
        semantic_type: currency
""")
    return tmp_path, yaml_path


class FakeLLM:
    """Deterministic embedder: just hashes characters into a fixed-dim vector."""
    async def complete(self, *args, **kwargs):
        raise NotImplementedError
    async def stream(self, *args, **kwargs):
        raise NotImplementedError
    async def embed(self, texts):
        # 16-dim hash-based embeddings. Same input → same output. Good enough
        # for the test that revenue-by-region retrieves the revenue example.
        def vec(t):
            v = [0.0] * 16
            for c in t.lower():
                v[ord(c) % 16] += 1.0
            return v
        return [vec(t) for t in texts]


def test_context_builder_full_pipeline(fixture_data):
    data_dir, yaml_path = fixture_data
    cat = DuckDBCatalogProvider(data_dir=str(data_dir))
    yaml_prov = YAMLMetadataProvider(str(yaml_path))
    vec = ChromaVectorProvider(path=":memory:", collection="t")
    llm = FakeLLM()

    # Seed the example index.
    indexer = ExampleIndexer(llm, vec)
    examples = [
        ExampleSQL(
            question="What is total revenue by region?",
            sql="SELECT region, SUM(amount) FROM orders GROUP BY region",
        ),
        ExampleSQL(
            question="How many orders did each customer place?",
            sql="SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id",
        ),
    ]
    asyncio.run(indexer.index(examples))

    builder = ContextBuilder(
        catalog=cat,
        metadata_providers=[yaml_prov],
        llm=llm,
        vector=vec,
    )
    ctx = asyncio.run(
        builder.build(
            question="revenue by region last month",
            tables=["main.orders"],
            top_k_examples=1,
        )
    )

    # Physical schema present.
    assert "main.orders" in ctx.tables
    t = ctx.tables["main.orders"]
    assert {c.name for c in t.columns} == {"order_id", "customer_id", "amount", "region"}

    # YAML enrichment applied.
    assert t.description == "Customer orders."
    amount_col = next(c for c in t.columns if c.name == "amount")
    assert amount_col.semantic_type == "currency"

    # Metadata sources tracked.
    assert "duckdb_catalog" in t.metadata_sources
    assert "yaml" in t.metadata_sources

    # Right example retrieved.
    assert len(ctx.retrieved_examples) == 1
    assert "revenue" in ctx.retrieved_examples[0].question.lower()
```

Run:

```bash
pytest tests/test_context_builder.py -v
```

Expected output:

```
tests/test_context_builder.py::test_context_builder_full_pipeline PASSED
```

If the test passes, you have built a working knowledge layer. The
agent in Module 6 will receive `ContextPackage` objects shaped like
this and reason from them.

### Definition of done

- All seven checkpoints complete.
- The integration test passes.
- `grep -rn "import openai\|import anthropic\|import duckdb\|import chromadb" agent/`
  shows these imports **only inside `providers/`**. Your
  `knowledge/` modules import provider *interfaces*, never SDKs.
- Commit: `git commit -m "module-04: knowledge layer with catalog,
  yaml metadata, and example retrieval"`.

### Common pitfalls

1. **The Chroma collection persists across test runs.** Either use
   `path=":memory:"` (preferred for tests) or call
   `vector._coll.delete()` in a teardown.

2. **`yaml.safe_load` returns `None` for an empty file.** Always
   default to `{}`: `self._raw = yaml.safe_load(f) or {}`.

3. **`enrich()` mutates `TableMeta` in place — and your test
   accidentally relies on order.** If two providers both want to set
   the description, the *first* wins under "only set if currently
   empty" semantics. Make sure your priority order is the order you
   want.

4. **You add `MetadataConflict` recording — and then realize you have
   nowhere to put it.** Punt. Either add `conflicts: list[str]` to
   `TableMeta` now, or leave conflict tracking for the stretch goal.
   Don't drop it silently; that defeats the purpose.

5. **`asyncio.run` inside a test is fine for now.** When you start
   building real async tests, switch to `pytest-asyncio`. Don't try
   to do both at once.

## Stretch

1. **Add a `RoomConfigOverridesMetadataProvider`.** It takes a
   `RoomConfig` (define a minimal version) and applies per-column
   overrides at the highest priority. Now you have a three-provider
   stack: catalog → yaml → room config. Test that the room override
   wins when both yaml and the override set the same column's
   description.

2. **Track `MetadataConflict` properly.** When the yaml provider
   overrides a non-empty scalar, append a conflict record to
   `TableMeta` (or to `ContextPackage`). Write a test that asserts
   the record contains the previous value and the source name.

3. **(Databricks mode)** Implement
   `agent/providers/databricks_catalog.py` against Unity Catalog. The
   interface is identical to `DuckDBCatalogProvider`; only the
   implementation differs. Run the same `test_context_builder_full_pipeline`
   against a real UC table. If you set up the test data carefully,
   the test passes unchanged.

## Reflection

1. The metadata stack has a clear priority order (catalog → yaml →
   room overrides). What happens if a room author wants to *remove*
   a synonym that the YAML adds? Sketch the API. Is "list extension
   only" the right rule, or should there be a way to subtract?

2. Your `ContextPackage` has four fields. Tiri's has fifteen-ish. Pick
   one Tiri field you didn't add and write down: when would your
   agent need it, and what would adding it cost? (This is how you
   decide whether to grow your model.)

3. The integration test uses a fake `embed` that hashes characters.
   It works because the words "revenue" and "region" appear in both
   the question and the matching example. What's the failure mode of
   this fake against a real test case where the question and the
   example share *concepts* but not *words*? (Hint: this is why real
   embeddings are worth their cost. The fake is a stand-in, not a
   substitute.)
