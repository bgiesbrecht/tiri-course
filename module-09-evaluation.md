# Module 9 — Evaluation as a first-class discipline

> **The rule:** If you can't measure correctness, you don't have an
> agent. Benchmarks are the spec. 100% or it's a bug.

In this module you'll build a benchmark suite for your room, run it,
and debug failures. By the end you'll have a score, a way to track
it over time, and — most importantly — the muscle memory for the
inner loop that turns "this question doesn't work" into "this
question works, and here's the test that proves it."

You'll also build a real `SynthesisAgent` (replacing the
`_synthesize_one_line` stand-in from Module 7), with the
causal-language regex scan that Module 6 introduced.

This is the module that separates "I built an agent" from "I built
an agent that works."

## Read

1. **`tiri/docs/feedback.md`** — the whole thing. Pay attention to:
   - The benchmark dataclass: `question`, `expected_sql`,
     `expected_row_count`, optional `tolerance`.
   - The two-tier match: `sql_match` (canonical SQL equality) and
     `result_match` (row count within tolerance). Either passing
     counts as the benchmark passing.
   - The score: passed / total. The doc says 100% before extensions.
     The doc means it.

2. **`tiri/docs/tuning.md`** — read in full. This is the playbook for
   "the benchmark failed; now what?" It walks through every failure
   mode (wrong table picked, missing snippet, ambiguous question,
   broken prompt) and what to do about each.

3. **`tiri/tiri/feedback/benchmark_runner.py`** — read the `run()` and
   `_run_one()` methods. Notice how a benchmark failure is *not* an
   exception — it's a structured `BenchmarkResult` that says exactly
   what went wrong.

4. **`tiri/tiri/feedback/sql_normalize.py`** — skim. The
   normalization rules: lowercase outside of string literals,
   collapse whitespace, drop trailing semicolons. Worth understanding
   why each rule exists — the comments cover it.

## Concepts

### Benchmarks vs unit tests

A unit test asserts "this function produces this output for this
input." It runs in milliseconds against mocked dependencies. It is
deterministic.

A benchmark asserts "this question produces a correct answer." It
runs in seconds against a real LLM. It is approximately
deterministic (temperature=0 helps; model nondeterminism, prompt
caching, network latency, all add noise).

You need both. They serve different purposes:

| | Unit test | Benchmark |
|---|---|---|
| Tests | Code mechanics | End-to-end correctness |
| Runs in | Milliseconds | Seconds-to-minutes |
| Deterministic | Yes | Approximately |
| Cost | Free | LLM API costs |
| When run | Every commit | Before deploys, after prompt edits |
| Failure means | A code bug | A prompt, snippet, or example needs work |

Tiri's discipline: unit tests run in CI on every commit and must
pass at 100%. Benchmarks run on demand (and in a scheduled job)
and must *also* pass at 100% before a release. The "100% bar" is
not aspirational — it's the only bar that means anything. 95% means
"5% of the time the system gives a wrong answer to a question your
team already knows the right answer to." That's not a successful
agent.

> **Concept: Evaluation harness.** The code that runs a benchmark
> suite and produces a pass/fail score. Tiri's `BenchmarkRunner` is
> ~200 lines. The two parts that matter: (1) a deterministic way to
> compare generated SQL to expected SQL (normalization), and (2) a
> structured `BenchmarkResult` per benchmark so failures are
> diagnosable.

### Why SQL normalization is necessary

A model might produce:

```sql
SELECT region, SUM(amount) FROM orders GROUP BY region
```

And you might have written:

```sql
select region, sum(amount) from orders group by region;
```

These are the same query. A `==` comparison says they aren't. A
naive "ignore whitespace" comparison fails on `SUM(amount)` vs
`SUM(  amount  )`. A naive "lowercase everything" breaks string
literals (`WHERE region = 'NORTH'` becomes `WHERE region =
'north'`, which doesn't match the data).

The normalization rules in `sql_normalize.py`:

- Lowercase everything *outside* of single or double quotes.
- Collapse all runs of whitespace to a single space.
- Strip leading/trailing whitespace.
- Drop trailing semicolons.

Not a full SQL parser. A pragmatic 80/20. Two queries that
normalize to the same string are considered equal.

This is also why we have `result_match` as a fallback: even when the
SQL strings don't match (model wrote a slightly different but
equivalent query), the row count comparison catches that the answer
is correct. Either match passes the benchmark.

### Example engineering vs code engineering

When a benchmark fails, the bug is rarely in your Python code. The
Python code is well-tested and small. The bug is usually:

- A bad **example** in your room's index (the model copied a
  wrong pattern from a similar question's example).
- A missing **snippet** (the model had to write a formula from
  scratch that the room should have provided).
- An ambiguous **question** in the benchmark itself (humans wrote
  it two different ways; only one matches `expected_sql`).
- A vague **prompt** (the system prompt doesn't tell the model
  about a domain convention it needs).

This means the work flow when you're debugging a benchmark failure
is mostly *not* "edit Python." It's:

1. Run the failing benchmark.
2. Read the generated SQL. Compare to `expected_sql`. What's the
   delta?
3. Look at the example index. Is there a similar example with the
   right pattern? If yes, why didn't the model use it? If no, add
   one.
4. Look at the snippets. Is there a formula the model invented that
   should have been provided?
5. Look at the prompt. Is there an instruction missing?
6. Look at the question itself. Is it actually well-formed, or is
   it three questions in a trench coat?

Most fixes are one of those five. Tiri's `docs/tuning.md` is the
checklist.

This is "example engineering" — tuning the room by editing its
examples, snippets, and instructions. Not "code engineering" —
editing Python.

> **Concept: Deterministic vs non-deterministic LLM behavior.** At
> `temperature=0.0`, modern LLMs are *approximately* deterministic
> — the same prompt usually produces the same output. Usually.
> Prompt caching, model updates, hardware variations, and tokenizer
> quirks can all cause the same input to produce different output
> across runs. Benchmarks that pass 95% of the time on the same
> question are often flaky for this reason. Treat run-to-run
> variance as a signal to *tighten the prompt or add an example*,
> not as random noise to ignore.

### The synthesis layer (real this time)

Module 7's `_synthesize_one_line` was a placeholder: read the row
count, return one line. A real `SynthesisAgent`:

- Takes the question, the SQL, and the query result.
- Renders a synthesis prompt with all three.
- Calls the LLM for a plain-English summary.
- **Runs a regex scan** for banned causal phrases ("caused by",
  "due to", etc.).
- If the scan finds a banned phrase, re-prompts the model with the
  explicit constraint.
- Returns an `Answer` with the synthesized text plus evidence.

The regex scan is the load-bearing part. The prompt asks the model
not to use causal language. The model usually obeys. The regex scan
catches the times it doesn't. Belt *and* suspenders.

```python
_BANNED_CAUSAL = re.compile(
    r"\b(caused by|because of|due to|led to|result of|results from)\b",
    re.IGNORECASE,
)

if _BANNED_CAUSAL.search(synthesis_text):
    # Re-prompt with a sharper instruction.
    ...
```

This is the most important structural-enforcement pattern in the
course. The same shape applies wherever you have a hard correctness
rule the model can't be trusted to obey on its own.

## Lab

Seven checkpoints. The first three build the benchmark harness; the
next two build a real synthesis agent; the last two are the tuning
loop in action.

### Checkpoint 1 — The `Benchmark` dataclass

In `agent/data_models.py`:

```python
@dataclass
class Benchmark:
    """One question/expected-SQL pair for evaluation."""
    id: str
    question: str
    expected_sql: str = ""               # if empty, only row-count compared
    expected_row_count: int | None = None
    tolerance: int = 0                   # |actual - expected| <= tolerance


@dataclass
class BenchmarkResult:
    benchmark_id: str
    question: str
    expected_sql: str
    generated_sql: str | None
    sql_match: bool
    result_match: bool | None       # None when expected_row_count is None
    passed: bool
    error: str | None = None


@dataclass
class BenchmarkReport:
    room_id: str
    total: int
    passed: int
    failed: int
    score: float
    results: list[BenchmarkResult] = field(default_factory=list)
```

A benchmark passes if either `sql_match` is True *or* `result_match`
is True. The `passed` field is the OR. Tracking both lets you
debug: "sql_match=False, result_match=True" means the model wrote
different SQL that gave the right answer (often fine, sometimes
worth investigating). "sql_match=True, result_match=False" means the
SQL matched but the data changed (a real regression).

### Checkpoint 2 — SQL normalization

`agent/feedback/sql_normalize.py`:

```python
"""SQL normalization for benchmark comparison.

Rules:
  - Lowercase keywords and identifiers, but NOT contents of strings
    or quoted identifiers.
  - Collapse all whitespace (spaces, tabs, newlines) to a single space.
  - Strip leading/trailing whitespace.
  - Remove trailing semicolons.

Not a full SQL parser. Treats '...' and "..." as opaque, lowercases
everything else.
"""
from __future__ import annotations
import re


_WHITESPACE = re.compile(r"\s+")


def normalize_sql(sql: str) -> str:
    out = []
    i = 0
    n = len(sql)
    while i < n:
        ch = sql[i]
        # Handle string literals: pass through unchanged
        if ch in ("'", '"'):
            close = ch
            out.append(ch)
            i += 1
            while i < n:
                out.append(sql[i])
                if sql[i] == close and (i == 0 or sql[i-1] != "\\"):
                    i += 1
                    break
                i += 1
            continue
        out.append(ch.lower())
        i += 1

    lowered = "".join(out)
    collapsed = _WHITESPACE.sub(" ", lowered).strip()
    if collapsed.endswith(";"):
        collapsed = collapsed[:-1].strip()
    return collapsed
```

Tests in `tests/test_sql_normalize.py`:

```python
from agent.feedback.sql_normalize import normalize_sql


def test_normalize_lowercases_keywords():
    assert normalize_sql("SELECT region FROM orders") == "select region from orders"


def test_normalize_preserves_string_literals():
    sql = "SELECT * FROM orders WHERE region = 'NORTH'"
    assert "'NORTH'" in normalize_sql(sql)


def test_normalize_collapses_whitespace():
    a = normalize_sql("SELECT   region\n   FROM\n  orders")
    b = normalize_sql("SELECT region FROM orders")
    assert a == b


def test_normalize_drops_trailing_semicolon():
    assert normalize_sql("SELECT 1;") == "select 1"
    assert normalize_sql("SELECT 1") == "select 1"


def test_normalize_treats_equivalent_queries_as_equal():
    a = "SELECT region, SUM(amount) FROM orders GROUP BY region"
    b = "  select   region,   sum(amount)\n  from orders\n  group by region;  "
    assert normalize_sql(a) == normalize_sql(b)
```

Run:

```bash
pytest tests/test_sql_normalize.py -v
```

All five tests should pass.

### Checkpoint 3 — The `BenchmarkRunner`

`agent/feedback/benchmark_runner.py`:

```python
"""Run a benchmark suite against the engine. Produce a structured report."""
from __future__ import annotations
import logging

from agent.data_models import (
    Benchmark, BenchmarkResult, BenchmarkReport,
)
from agent.engine.room_engine import RoomEngine
from agent.feedback.sql_normalize import normalize_sql


_log = logging.getLogger("agent.benchmark")


class BenchmarkRunner:
    def __init__(self, engine: RoomEngine) -> None:
        self._engine = engine

    async def run(
        self, room_id: str, benchmarks: list[Benchmark]
    ) -> BenchmarkReport:
        results = []
        for bench in benchmarks:
            results.append(await self._run_one(room_id, bench))
        passed = sum(1 for r in results if r.passed)
        return BenchmarkReport(
            room_id=room_id,
            total=len(results),
            passed=passed,
            failed=len(results) - passed,
            score=(passed / len(results)) if results else 0.0,
            results=results,
        )

    async def _run_one(
        self, room_id: str, bench: Benchmark
    ) -> BenchmarkResult:
        conv_id = f"benchmark-{bench.id}"
        try:
            turn = await self._engine.chat(
                room_id=room_id,
                conversation_id=conv_id,
                question_text=bench.question,
            )
        except Exception as e:
            _log.exception("benchmark %s crashed", bench.id)
            return BenchmarkResult(
                benchmark_id=bench.id, question=bench.question,
                expected_sql=bench.expected_sql, generated_sql=None,
                sql_match=False, result_match=None, passed=False,
                error=str(e),
            )

        if turn.unable_to_answer or turn.clarification_question:
            return BenchmarkResult(
                benchmark_id=bench.id, question=bench.question,
                expected_sql=bench.expected_sql, generated_sql=None,
                sql_match=False, result_match=None, passed=False,
                error=(
                    turn.unable_to_answer or turn.clarification_question
                    or "no SQL generated"
                ),
            )

        # turn.answer is set; we need the SQL to compare. In our slim
        # engine the SQL isn't exposed in Answer — we'd need to add a
        # field. See Checkpoint 4 for that change.
        generated_sql = (turn.answer.metadata or {}).get("sql", "") \
            if turn.answer else ""

        sql_match = (
            bool(bench.expected_sql)
            and normalize_sql(bench.expected_sql) == normalize_sql(generated_sql)
        )
        result_match = None
        if bench.expected_row_count is not None and turn.answer:
            actual = (turn.answer.metadata or {}).get("row_count", -1)
            result_match = abs(actual - bench.expected_row_count) <= bench.tolerance

        passed = sql_match or (result_match is True)
        return BenchmarkResult(
            benchmark_id=bench.id, question=bench.question,
            expected_sql=bench.expected_sql, generated_sql=generated_sql,
            sql_match=sql_match, result_match=result_match,
            passed=passed,
        )
```

### Checkpoint 4 — Expose the SQL from the engine

For benchmarks to compare SQL, the engine needs to put the SQL in
the answer. Update `agent/data_models.py`:

```python
@dataclass
class Answer:
    text: str
    evidence: list[Evidence] = field(default_factory=list)
    caveats: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    # metadata can hold {"sql": "...", "row_count": 4, "attempts": 1}
```

Update `RoomEngine.chat()` to populate it (the final `Turn(answer=Answer(...))`
construction):

```python
return Turn(
    turn_id=turn_id,
    question=question,
    answer=Answer(
        text=summary,
        evidence=[Evidence(...)],
        caveats=_default_caveats(config),
        metadata={
            "sql": sql_result.sql,
            "row_count": query_result.row_count,
            "attempts": sql_result.attempts,
        },
    ),
)
```

This is a tiny widening of the dataclass — but it's worth thinking
about. The benchmark runner needed access to the generated SQL, so
we exposed it on `Answer.metadata`. Real systems exposing this on
the wire have an additional decision: do you put the SQL in the API
response? Tiri does — *every* answer includes the SQL it was based
on, because that's part of the trust mechanism. The benchmark
runner is just one consumer of that surface.

### Checkpoint 5 — Author benchmarks

Edit `rooms/sales-room.json` (or a sibling
`rooms/sales-room.benchmarks.json` — your choice). Add benchmarks
that match the room's tables and examples:

```json
{
  "benchmarks": [
    {
      "id": "b1-revenue-by-region",
      "question": "What was total revenue by region?",
      "expected_sql": "SELECT region, SUM(amount) FROM orders GROUP BY region",
      "expected_row_count": 4
    },
    {
      "id": "b2-customer-count",
      "question": "How many distinct customers ordered?",
      "expected_sql": "SELECT COUNT(DISTINCT customer_id) FROM orders",
      "expected_row_count": 1
    },
    {
      "id": "b3-largest-order",
      "question": "What was the largest single order amount?",
      "expected_sql": "SELECT MAX(amount) FROM orders",
      "expected_row_count": 1
    },
    {
      "id": "b4-orders-per-customer",
      "question": "How many orders did each customer place?",
      "expected_sql": "SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id",
      "expected_row_count": 3
    },
    {
      "id": "b5-refusal-causes",
      "question": "Why did revenue drop last quarter?",
      "expected_sql": "",
      "expected_row_count": null
    }
  ]
}
```

The fifth benchmark is interesting. It has no `expected_sql` —
it's testing that the agent *refuses* to answer (or asks for
clarification). You can extend `Benchmark` with an
`expected_refusal: bool = False` field and update the runner to
honor it. (Stretch goal.)

Author at least five benchmarks for your room. Make sure they cover:

- A simple aggregation (revenue by group).
- An aggregation with a different shape (count distinct, max).
- A multi-step or filtered question.
- An edge case you specifically want to catch (NULLs, time
  filtering, a specific column the model has gotten wrong before).
- A question the agent should refuse.

### Checkpoint 6 — Run it

`scripts/bench.py`:

```python
"""Run benchmarks for a room. Print a report."""
import asyncio
import json
import sys

from agent.config import Config
from agent.container import build_container
from agent.data_models import Benchmark
from agent.feedback.benchmark_runner import BenchmarkRunner


async def main(room_id: str, benchmarks_path: str):
    cfg = Config.load()
    container = build_container(cfg)
    engine = container["engine"]

    with open(benchmarks_path) as f:
        raw = json.load(f)
    benchmarks = [Benchmark(**b) for b in raw["benchmarks"]]

    runner = BenchmarkRunner(engine)
    report = await runner.run(room_id, benchmarks)

    print(f"\n── Benchmark Report: {report.room_id} ──")
    print(f"  Score: {report.score:.1%}  ({report.passed}/{report.total})")
    print(f"")
    for r in report.results:
        status = "PASS" if r.passed else "FAIL"
        print(f"  [{status}] {r.benchmark_id}: {r.question}")
        if not r.passed:
            print(f"          expected: {r.expected_sql!r}")
            print(f"          got:      {r.generated_sql!r}")
            print(f"          sql_match={r.sql_match}, result_match={r.result_match}")
            if r.error:
                print(f"          error: {r.error}")

    sys.exit(0 if report.score == 1.0 else 1)


if __name__ == "__main__":
    asyncio.run(main(sys.argv[1], sys.argv[2]))
```

Run:

```bash
python scripts/bench.py sales-room rooms/sales-room.benchmarks.json
```

Expected output (your numbers will vary):

```
── Benchmark Report: sales-room ──
  Score: 60.0%  (3/5)

  [PASS] b1-revenue-by-region: What was total revenue by region?
  [PASS] b2-customer-count: How many distinct customers ordered?
  [FAIL] b3-largest-order: What was the largest single order amount?
          expected: 'SELECT MAX(amount) FROM orders'
          got:      'SELECT MAX(amount) AS max_amount FROM orders'
          sql_match=False, result_match=True
  [FAIL] b4-orders-per-customer: How many orders did each customer place?
          expected: 'SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id'
          got:      'SELECT customer_id, COUNT(*) AS n_orders FROM orders GROUP BY customer_id ORDER BY n_orders DESC'
          sql_match=False, result_match=True
  [PASS] b5-refusal-causes: Why did revenue drop last quarter?
```

**Your first benchmark report.** Read the failures carefully. The
two "fails" above are *not* genuinely wrong — the model added a
column alias (`AS max_amount`) and an `ORDER BY` clause. The SQL
doesn't match the expected, but the results do. These are the
ambiguous-pass cases the tuning module is for.

If your score is below 100%, walk the tuning loop:

1. **Did the model add an alias or ordering that's harmless?** Loosen
   your `expected_sql` (use the model's actual output if it's
   correct) or just rely on `result_match`.

2. **Did the model use a different table or formula?** Check whether
   your room's examples teach the right pattern. Add an example.

3. **Did the model refuse a question that should have been
   answerable?** Check whether your room's instruction covers this
   class of question. Refine it.

4. **Did the model invent a column?** Check whether the metadata has
   the right column descriptions. Add them.

Re-run after each change. The goal is 100% on every run. Anything
less is a real bug you haven't isolated yet — flakiness, ambiguous
benchmark, missing example, wrong prompt.

### Checkpoint 7 — Build the real `SynthesisAgent`

Now that we have benchmarks, we can replace the stand-in synthesis
with a real one — and benchmark *that* too.

`agent/engine/prompt_templates/synthesis.txt`:

```
You are an analyst. Summarize the result of a SQL query in plain
English for a non-technical reader.

## Question
{question}

## SQL that produced the result
{sql}

## Result
{result_table}

Write a one-paragraph summary of what the data shows. Be specific
about numbers. Be concise.

You MUST NOT use any of the following phrases:
- "caused by"
- "because of"
- "due to"
- "led to"
- "result of"

These phrases imply causation that data cannot prove. Use
alternatives like "associated with", "coincided with", or "occurred
alongside" if you need to describe a relationship between numbers.

Return only the summary. No headers, no preamble.
```

`agent/engine/agents/synthesis_agent.py`:

```python
"""SynthesisAgent — produces a plain-English summary with banned-phrase scan."""
from __future__ import annotations
import logging
import re

from agent.data_models import LLMMessage, QueryResult
from agent.providers.base import LLMProvider, LLMProviderError
from agent.engine.prompts import load_template, render


_TEMPLATE = load_template("synthesis.txt")
_log = logging.getLogger("agent.synthesis")

_BANNED = re.compile(
    r"\b(caused by|because of|due to|led to|result(s)? (from|of))\b",
    re.IGNORECASE,
)


class SynthesisAgent:
    def __init__(self, llm: LLMProvider, max_retries: int = 2) -> None:
        self._llm = llm
        self._max_retries = max_retries

    async def run(
        self, question: str, sql: str, query_result: QueryResult,
    ) -> str:
        result_table = _format_result(query_result)
        prompt = render(_TEMPLATE,
                        question=question, sql=sql, result_table=result_table)

        messages = [LLMMessage(role="system", content=prompt)]
        for attempt in range(self._max_retries):
            try:
                response = await self._llm.complete(messages, task="synthesis")
            except LLMProviderError:
                # Degrade: return a mechanical summary.
                return _format_mechanical(query_result)

            text = response.content.strip()
            if not _BANNED.search(text):
                return text

            # Banned phrase found. Re-prompt with the specific phrase.
            _log.info(
                "synthesis attempt %d had banned causal phrase; re-prompting",
                attempt + 1,
            )
            messages.append(LLMMessage(role="assistant", content=text))
            messages.append(LLMMessage(
                role="user",
                content=(
                    "Your response used a banned causal phrase. Rewrite "
                    "the summary without any of: 'caused by', 'because of', "
                    "'due to', 'led to', 'result of'. Use 'associated with' "
                    "or 'coincided with' if you need to describe a "
                    "relationship between numbers."
                ),
            ))

        # Exhausted retries — fall back to mechanical summary rather
        # than serve a banned phrase to the user.
        _log.warning("synthesis exhausted retries; falling back to mechanical")
        return _format_mechanical(query_result)


def _format_result(qr: QueryResult) -> str:
    """Format the query result as a small text table."""
    if not qr.rows:
        return "(no rows)"
    lines = [" | ".join(qr.columns)]
    lines.append("-" * len(lines[0]))
    for row in qr.rows[:20]:
        lines.append(" | ".join(str(c) for c in row))
    if qr.row_count > 20:
        lines.append(f"... ({qr.row_count - 20} more rows)")
    return "\n".join(lines)


def _format_mechanical(qr: QueryResult) -> str:
    if qr.row_count == 0:
        return "The query returned no rows."
    if qr.row_count == 1 and len(qr.columns) == 1:
        return f"{qr.columns[0]}: {qr.rows[0][0]}"
    return (
        f"The query returned {qr.row_count} rows across "
        f"{len(qr.columns)} columns: {', '.join(qr.columns)}."
    )
```

Tests for the banned-phrase enforcement, in
`tests/test_synthesis_agent.py`:

```python
import pytest
from agent.data_models import QueryResult, LLMResponse
from agent.engine.agents.synthesis_agent import SynthesisAgent
from agent.providers.base import LLMProvider


class FakeLLM(LLMProvider):
    def __init__(self, queued: list[str]) -> None:
        self._queue = list(queued)
        self.call_count = 0
    async def complete(self, messages, **kwargs):
        self.call_count += 1
        return LLMResponse(content=self._queue.pop(0))
    async def stream(self, *a, **kw): raise NotImplementedError
    async def embed(self, *a, **kw): raise NotImplementedError


def _qr():
    return QueryResult(
        columns=["region", "revenue"],
        rows=[["NORTH", 1200000], ["SOUTH", 800000]],
        row_count=2,
    )


@pytest.mark.asyncio
async def test_synthesis_passes_clean_text():
    llm = FakeLLM(["North led with $1.2M; South followed with $800K."])
    agent = SynthesisAgent(llm)
    text = await agent.run("revenue by region", "SELECT ...", _qr())
    assert "North led" in text
    assert llm.call_count == 1


@pytest.mark.asyncio
async def test_synthesis_reprompts_on_banned_phrase():
    llm = FakeLLM([
        "North led; growth was caused by the holiday season.",   # has 'caused by'
        "North led with $1.2M; South followed with $800K.",      # clean
    ])
    agent = SynthesisAgent(llm)
    text = await agent.run("revenue by region", "SELECT ...", _qr())
    assert "caused by" not in text.lower()
    assert llm.call_count == 2


@pytest.mark.asyncio
async def test_synthesis_falls_back_to_mechanical_on_persistent_violation():
    llm = FakeLLM([
        "Growth caused by promotions.",
        "Increase due to seasonality.",
        # max_retries=2, so we exhaust here.
    ])
    agent = SynthesisAgent(llm, max_retries=2)
    text = await agent.run("revenue by region", "SELECT ...", _qr())
    # Mechanical fallback, not the banned content.
    assert "caused by" not in text.lower()
    assert "due to" not in text.lower()
    assert "rows" in text.lower()
```

Run:

```bash
pytest tests/test_synthesis_agent.py -v
```

All three tests should pass. The second test is the most important
— it's the structural enforcement working as designed.

Now wire it into the engine. In `agent/engine/room_engine.py`,
replace `_synthesize_one_line` with the real agent:

```python
# In RoomEngine.chat(), step 6:
synthesis_agent = SynthesisAgent(self._llm)
summary = await synthesis_agent.run(
    question_text, sql_result.sql, query_result
)
```

Re-run your benchmarks:

```bash
python scripts/bench.py sales-room rooms/sales-room.benchmarks.json
```

The answers should now read as proper paragraphs, not mechanical
summaries. Verify by hand: do the synthesized texts read like a
careful analyst wrote them? Are any using banned phrases? (They
shouldn't — the regex catches them — but read a few to confirm.)

### Definition of done

- `agent/data_models.py` has `Benchmark`, `BenchmarkResult`,
  `BenchmarkReport`.
- `agent/feedback/sql_normalize.py` and `benchmark_runner.py`
  implemented and tested.
- `rooms/sales-room.benchmarks.json` has at least five benchmarks
  covering aggregation, edge cases, and a refusal.
- `scripts/bench.py` produces a report and exits non-zero if the
  score isn't 100%.
- `agent/engine/agents/synthesis_agent.py` implements the real
  synthesis with banned-phrase enforcement.
- Synthesis tests pass.
- **Your benchmark score is 100%.** If it isn't, walk the tuning
  loop until it is, or document each failure in
  `rooms/sales-room.known-issues.md` with a one-line "why this is
  not a real bug" explanation. Either is acceptable; "I'll get to
  it later" is not.
- Commit:
  ```bash
  git add . && git commit -m "module-09: benchmarks + real synthesis with banned-phrase scan"
  ```

### Common pitfalls

1. **You write benchmarks that the model can pass by guessing.**
   "What is the SUM of orders?" is too vague — the model might
   guess `SUM(amount)`, `SUM(order_id)`, or `COUNT(*)`. Make the
   question specific: "What is the SUM of the amount column from
   the orders table?"

2. **Your `expected_sql` is too strict.** A query that adds `AS
   total` or `ORDER BY ...` to a passing query is still correct.
   Either accept multiple equivalent forms (run normalization both
   ways and check), or rely on `result_match`.

3. **Your benchmarks are non-deterministic across runs.** Same
   question, sometimes passes, sometimes fails. Causes: prompt
   caching, model variance, ambiguous question. The fix is *not*
   to retry; it's to make the question and the prompt unambiguous
   enough that the model produces the same answer every time.

4. **You skip authoring a refusal benchmark.** Refusals are the
   most important behaviors to test — they're what makes the
   system trustworthy. If you don't benchmark refusals, your agent
   will silently regress into answering questions it should refuse.

5. **You "tune" by lowering the benchmark's expectations.** If a
   benchmark fails because the model picks a different (wrong)
   table, the fix is *not* to relax the expected SQL. The fix is
   to make the room teach the right pattern. Relaxing benchmarks
   to make them pass is how you ship a 99% agent that's right 50%
   of the time.

6. **Benchmarks fail because the LLM API rate-limited you.** Sleep
   a fraction of a second between benchmarks if you see this
   happening, or batch them and back off. Don't increase
   `max_retries` — that masks the issue.

## Stretch

1. **Add `expected_refusal: bool` to `Benchmark`.** Update the
   runner to check that the turn has `unable_to_answer` set when
   `expected_refusal=True`. Add the b5 benchmark from above to
   actually test this.

2. **CSV/JSON report output.** `scripts/bench.py --output
   results.json` writes the report to a file. Check the file into
   git or upload it to a bucket. Now you can plot score over time.

3. **Run benchmarks against multiple LLM backends.** Use the
   per-call `model` parameter on `LLMProvider` (the one we added
   in Module 2) to score the same room against, say,
   `gpt-4o-mini` and `claude-haiku-4-5`. Compare results.
   Different models will fail different benchmarks, and where they
   *agree on a failure* is usually a real bug in your room.

4. **The `_format_result` table is naive.** For wide tables (>5
   columns) it gets unreadable. Build a smarter formatter — maybe
   only the first 5 columns plus a count of how many were
   truncated. Real Tiri does this in `tiri/engine/agents/viz_agent.py`'s
   formatter helpers.

## Reflection

1. The benchmark score is 100% on a 5-benchmark suite. Is that
   meaningful evidence the room works? What about on a 50-benchmark
   suite? At what size does the score become a real claim about the
   room?

2. The synthesis agent has a two-attempt retry loop for banned
   phrases. Tiri has three. What's the right number for your
   domain? When would you tighten it (fewer retries, more
   mechanical fallbacks)? When would you loosen it?

3. The "tuning loop" mostly edits the room (examples, snippets,
   prompts), not Python. Is this a bug or a feature? What does it
   imply about who can productively maintain the agent — only
   engineers, or also analysts and domain experts? What would a
   "tuning UI" look like that let a non-engineer iterate?

4. Your refusal benchmark (b5) is a single test. In production,
   how would you grow refusal testing into a discipline — say, ten
   distinct refusal cases that exercise different rules from your
   manifesto? Sketch the list.
