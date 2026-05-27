# Module 6 — The compound agent pipeline

> **The rule:** Many small focused agents > one omniscient prompt.
> Each agent does one thing and is testable in isolation.

This is the module that ties everything together. By the end you'll
have two real agents — `IntentAgent` and `SQLAgent` — that take a
`ContextPackage` and a question, talk to the LLM, and return typed
results. They are the first pieces of the system that actually
*reason*. Everything before this was infrastructure for these.

This is the longest module so far. The concepts are dense; the
labs take time. Plan for ~6–8 hours; do it in two sittings.

## Read

1. **`tiri/docs/agents.md`** — the whole thing. This is the
   specification you're implementing. Pay attention to:
   - The list of agents (IntentAgent, ClarifyAgent, SQLAgent,
     VizAgent, SynthesisAgent, PlanningAgent, HypothesisAgent) and
     what each one returns.
   - The "Test cases" tables — every row is a MUST you should be
     able to write a test for.

2. **`tiri/tiri/engine/agents/intent_agent.py`** — read in full.
   Notice:
   - It's small. ~120 lines including helpers.
   - One LLM call per `run()`. No loops, no fallbacks beyond JSON
     parsing.
   - The output is a typed `IntentResult`, not a dict.
   - The JSON parser tolerates a few common model failure modes
     (wrapped in fences, trailing prose). It does *not* try to
     repair invalid JSON.

3. **`tiri/tiri/engine/agents/sql_agent.py`** — read in full.
   Notice:
   - The retry loop. Up to `max_retries` attempts. On each failure,
     the validator's error message is appended to the conversation
     and the model gets another try.
   - Markdown fence stripping (the model sometimes wraps SQL in
     ` ```sql ... ``` ` despite being told not to).
   - The "CANNOT_ANSWER:" escape hatch — the model can decline
     explicitly, and that's a valid return value, not a failure.

4. **`tiri/tiri/engine/agents/synthesis_agent.py`** — skim. Look for
   the regex scan after the LLM responds. That's the causal-language
   enforcement: prompt asks the model not to use "caused by", and
   the agent checks afterward. Belt and suspenders.

5. **`tiri/tiri/engine/agents/viz_agent.py`** — skim. Notice that
   the chart spec is built in *Python*, not by the LLM. The LLM is
   used only for the one-sentence summary.

## Concepts

### What an agent is, concretely

An agent, in this course, is a small unit of code with this shape:

1. Takes typed input (a `ContextPackage`, a question, maybe a prior
   result from another agent).
2. Renders one or more prompt templates using the inputs.
3. Calls the LLM (`llm.complete(...)` — usually one call, sometimes
   a retry loop).
4. Parses the response into a typed result.
5. Optionally runs a post-generation check (regex scan, JSON
   validation, SQL validation).
6. Returns the typed result.

That's it. An agent is *not* a class hierarchy, an event loop, a
state machine, or a "personality." It's a function with a prompt
attached.

The compound agent pipeline is just a sequence of these functions,
each one consuming the previous one's output:

```python
# Sketch — Module 7 actually wires this.
intent = await intent_agent.run(question, context)
if intent.kind == "answerable":
    sql_result = await sql_agent.run(question, context, intent)
    if sql_result.is_valid:
        rows = await query.execute(sql_result.sql)
        answer = await synthesis_agent.run(question, rows, context)
        return Turn(answer=answer, ...)
elif intent.kind == "clarification_needed":
    return Turn(clarification_question=intent.clarification, ...)
else:
    return Turn(unable_to_answer=intent.refusal_reason, ...)
```

This is deliberately boring. The control flow is in plain Python:
`if`, `await`, `return`. The LLM is not driving the flow; the
*engine* is. Each agent's only job is to do its small piece well.

### Why decomposition wins for high-stakes systems

The case for "one big prompt that does everything":

- Fewer LLM calls. Cheaper. Faster.
- Less code. One prompt, one parser, one call.
- The model has all the context at once and can "reason" across it.

The case for many small agents:

- **Each piece is testable in isolation.** You can write a fixture
  `ContextPackage` and assert `IntentAgent` returns
  `kind="clarification_needed"` for a vague question, without ever
  running SQL.
- **Failure modes localize.** When something goes wrong, you can see
  *which agent* produced the wrong output, instead of debugging a
  monolithic response.
- **You can swap models per task.** A small/cheap model is fine for
  intent classification; a bigger model may be worth the cost for
  SQL generation. With one prompt that's impossible; with multiple
  agents it's a config change (Module 10).
- **Prompts get small and focused.** A small prompt has less for the
  model to misinterpret. The biggest cause of bad LLM output is
  putting too much in one prompt.

For low-stakes systems (a chatbot that helps you book a restaurant,
a "write me a poem about cats" demo), the costs of decomposition
outweigh the benefits. For systems that need to be defensible — the
kind this course is about — decomposition is the only approach that
makes the system testable, debuggable, and tunable.

> **Concept: Tool use, revisited.** Some LLM frameworks build the
> pipeline by having the LLM itself decide which agent (or "tool")
> to call next, using the model's function-calling API. That's the
> "single-prompt agent with tools" pattern. It's flexible and
> popular. But it puts the *control flow* inside the model — which
> means you can't test the control flow without running the model.
> Tiri's pattern keeps the control flow in code. Each agent is a
> tool, but the engine decides what runs when. Both patterns are
> valid; the one this course uses is the one that survives
> contact with high-stakes correctness requirements.

### Structured output: the JSON contract

Most agents in Tiri ask the LLM to return JSON in a specific shape.
This is the *contract* between the agent and the model:

```
Return your answer as JSON with exactly this shape:
{
  "intent": "answerable" | "clarification_needed" | "refused",
  "reasoning": "<one sentence>",
  ...
}
```

Three things make this contract robust:

1. **Show the exact shape.** Models do much better at producing JSON
   when shown what valid JSON for *this specific task* looks like.
2. **Use enum-style strings, not freeform.** `"intent":
   "answerable"` is checkable; `"intent": "this is an answerable
   question"` isn't.
3. **Parse defensively, but don't repair.** Strip markdown fences,
   tolerate trailing prose, but if the JSON itself is malformed,
   raise. Trying to "fix" broken JSON from the model is how silent
   corruption happens.

> **Concept: JSON mode.** Most LLM APIs now have a "JSON mode" or
> "structured output" feature that guarantees the response is valid
> JSON. OpenAI's `response_format={"type": "json_object"}` is the
> simplest version. Anthropic's tool use with a JSON schema is more
> powerful. Use these when you have them; they reduce the
> defensive-parsing surface area. The lab will work without them,
> using a regex-and-parse approach, because not every model
> supports them.

### The validate-and-retry pattern

The SQL agent introduces a pattern you'll use again: **let the
validator's error message be part of the next prompt**.

```python
attempt = 0
while attempt < self._max_retries:
    attempt += 1
    response = await self._llm.complete(messages)
    sql = strip_markdown_fences(response.content)
    is_valid, error = await self._query.validate(sql)
    if is_valid:
        return SQLResult(is_valid=True, sql=sql, attempts=attempt)
    # Feed the validator's error back into the conversation.
    messages.append(LLMMessage(role="assistant", content=sql))
    messages.append(LLMMessage(
        role="user",
        content=f"That SQL has an error: {error}\nPlease fix it.",
    ))
```

This works because validators produce *useful* error messages
("column foo.bar does not exist" / "ambiguous reference 'amount'"),
and modern LLMs are very good at applying those error messages to
their previous output. Three attempts usually resolves all but the
hardest cases. Beyond three, more retries rarely help — the model is
stuck on a misunderstanding that more iterations won't fix.

The pattern generalizes: any time an agent produces something
*checkable* (SQL, JSON, code, structured prose), you can put the
checker's error message into the next attempt's prompt. It's one of
the highest-ROI patterns in agentic systems.

### Structural enforcement, not just prompts

Tiri's vision (Module 0) forbids causal language in synthesis: no
"caused by", no "due to", no "led to." The prompt asks the model not
to use them. The model usually obeys. *Usually.*

The synthesis agent runs a regex scan over the model's output and
re-prompts if a banned phrase appears:

```python
_BANNED = re.compile(
    r"\b(caused by|due to|because of|led to|result of)\b",
    re.IGNORECASE,
)

if _BANNED.search(response.content):
    # Re-prompt with a more specific instruction.
    ...
```

Two reasons this is non-negotiable:

1. **Prompts are advisory.** A model that obeys "don't use 'caused
   by'" 99 times out of 100 will eventually fail. For a system whose
   thesis is "the agent doesn't bluff", a 1% bluff rate is a
   failure.

2. **Structural enforcement is testable.** You can write a test that
   feeds the synthesis agent a context that *invites* causal
   language, and assert the output passes the regex. Prompt-only
   enforcement isn't testable in the same way.

The same pattern applies to anywhere your system has a hard
correctness rule. Prompts request; code enforces.

## Lab

You'll build two agents — `IntentAgent` and `SQLAgent` — and test
each in isolation against fixture `ContextPackage` objects. Seven
checkpoints. Plan for two sittings.

### Checkpoint 1 — Result dataclasses

In `agent/data_models.py`, append:

```python
@dataclass
class IntentResult:
    """What IntentAgent returns."""
    kind: Literal["answerable", "clarification_needed", "refused"]
    reasoning: str = ""
    clarification: str = ""           # populated when kind == "clarification_needed"
    refusal_reason: str = ""          # populated when kind == "refused"
    relevant_tables: list[str] = field(default_factory=list)


@dataclass
class SQLResult:
    """What SQLAgent returns."""
    is_valid: bool
    sql: str = ""
    attempts: int = 0
    error: str | None = None
    explanation: str = ""
```

### Checkpoint 2 — IntentAgent skeleton

Create `agent/engine/agents/` and the file:

```bash
mkdir -p agent/engine/agents
touch agent/engine/agents/__init__.py
touch agent/engine/agents/intent_agent.py
```

`agent/engine/agents/intent_agent.py`:

```python
"""IntentAgent — classifies a question and routes it."""
from __future__ import annotations
import json
import re
import logging

from agent.data_models import (
    ContextPackage, IntentResult, LLMMessage,
)
from agent.providers.base import LLMProvider, LLMProviderError
from agent.engine.prompts import load_template, render


_TEMPLATE = load_template("intent_classification.txt")
_log = logging.getLogger("agent.intent")


class IntentAgent:
    def __init__(self, llm: LLMProvider) -> None:
        self._llm = llm

    async def run(
        self, question: str, context: ContextPackage
    ) -> IntentResult:
        prompt = render(
            _TEMPLATE,
            table_descriptions=_format_tables(context),
            refused_claim_types=_format_refusals(context),
            text_instruction="(none)",        # we'll wire this in Module 7
            example_questions=_format_examples(context),
            question=question,
        )
        try:
            response = await self._llm.complete(
                [LLMMessage(role="system", content=prompt)],
                task="intent",
            )
        except LLMProviderError as e:
            _log.warning("IntentAgent LLM call failed: %s", e)
            raise

        raw = _parse_json(response.content)
        return _to_intent_result(raw)


def _format_tables(ctx: ContextPackage) -> str:
    """Render the available tables for the prompt."""
    lines = []
    for full_name, meta in ctx.tables.items():
        desc = meta.description or "(no description)"
        lines.append(f"- {full_name}: {desc}")
    return "\n".join(lines) if lines else "(no tables)"


def _format_refusals(ctx: ContextPackage) -> str:
    """List of refused claim types from the room config (Module 7)."""
    # TODO: when RoomConfig lands in Module 7, pull from ctx.refused_claims.
    return "(none)"


def _format_examples(ctx: ContextPackage) -> str:
    if not ctx.retrieved_examples:
        return "(none)"
    return "\n".join(
        f"- Q: {e.question}" for e in ctx.retrieved_examples[:3]
    )


def _parse_json(content: str) -> dict:
    """Tolerate fences and trailing prose; do not repair invalid JSON."""
    stripped = content.strip()
    if stripped.startswith("```"):
        stripped = re.sub(r"^```(?:json)?\s*", "", stripped)
        stripped = re.sub(r"\s*```\s*$", "", stripped)
    try:
        return json.loads(stripped)
    except json.JSONDecodeError:
        match = re.search(r"\{.*\}", stripped, re.DOTALL)
        if not match:
            raise LLMProviderError(
                f"IntentAgent response is not valid JSON: {content!r}"
            )
        try:
            return json.loads(match.group(0))
        except json.JSONDecodeError as e:
            raise LLMProviderError(
                f"IntentAgent JSON parse failed: {e}; content={content!r}"
            ) from e


def _to_intent_result(raw: dict) -> IntentResult:
    kind = raw.get("intent")
    if kind not in ("answerable", "clarification_needed", "refused"):
        raise LLMProviderError(f"unknown intent kind: {kind!r}")
    return IntentResult(
        kind=kind,
        reasoning=raw.get("reasoning", ""),
        clarification=raw.get("clarification", ""),
        refusal_reason=raw.get("refusal_reason", ""),
        relevant_tables=raw.get("relevant_tables", []) or [],
    )
```

### Checkpoint 3 — Test IntentAgent in isolation

`tests/test_intent_agent.py`:

```python
import json
import pytest
from agent.data_models import ContextPackage, TableMeta, LLMMessage, LLMResponse
from agent.engine.agents.intent_agent import IntentAgent
from agent.providers.base import LLMProvider


class FakeLLM(LLMProvider):
    """A fake LLM that returns a queued response. No network."""
    def __init__(self, queued_response: str) -> None:
        self._response = queued_response

    async def complete(self, messages, **kwargs):
        return LLMResponse(content=self._response)

    async def stream(self, messages, **kwargs):
        raise NotImplementedError

    async def embed(self, texts):
        raise NotImplementedError


def _ctx(tables=None) -> ContextPackage:
    return ContextPackage(
        question="",
        tables=tables or {
            "main.orders": TableMeta(
                full_name="main.orders",
                description="Customer orders, one row per order.",
            )
        },
        retrieved_examples=[],
        schema_descriptions={},
    )


@pytest.mark.asyncio
async def test_intent_answerable():
    fake = FakeLLM(json.dumps({
        "intent": "answerable",
        "reasoning": "revenue is in the orders table",
        "relevant_tables": ["main.orders"],
    }))
    agent = IntentAgent(fake)
    result = await agent.run("revenue by region", _ctx())
    assert result.kind == "answerable"
    assert result.relevant_tables == ["main.orders"]


@pytest.mark.asyncio
async def test_intent_clarification_needed():
    fake = FakeLLM(json.dumps({
        "intent": "clarification_needed",
        "reasoning": "the question is ambiguous",
        "clarification": "Which time period did you mean?",
    }))
    agent = IntentAgent(fake)
    result = await agent.run("what's the revenue?", _ctx())
    assert result.kind == "clarification_needed"
    assert "time period" in result.clarification


@pytest.mark.asyncio
async def test_intent_refused():
    fake = FakeLLM(json.dumps({
        "intent": "refused",
        "reasoning": "asks for a causal claim",
        "refusal_reason": "I cannot determine why revenue changed.",
    }))
    agent = IntentAgent(fake)
    result = await agent.run("why did revenue drop?", _ctx())
    assert result.kind == "refused"
    assert "cannot determine" in result.refusal_reason


@pytest.mark.asyncio
async def test_intent_tolerates_markdown_fences():
    fake = FakeLLM(
        "```json\n"
        + json.dumps({"intent": "answerable"})
        + "\n```"
    )
    agent = IntentAgent(fake)
    result = await agent.run("revenue", _ctx())
    assert result.kind == "answerable"


@pytest.mark.asyncio
async def test_intent_raises_on_invalid_json():
    from agent.providers.base import LLMProviderError
    fake = FakeLLM("not json at all")
    agent = IntentAgent(fake)
    with pytest.raises(LLMProviderError):
        await agent.run("revenue", _ctx())
```

Run:

```bash
pytest tests/test_intent_agent.py -v
```

Expected output:

```
tests/test_intent_agent.py::test_intent_answerable PASSED
tests/test_intent_agent.py::test_intent_clarification_needed PASSED
tests/test_intent_agent.py::test_intent_refused PASSED
tests/test_intent_agent.py::test_intent_tolerates_markdown_fences PASSED
tests/test_intent_agent.py::test_intent_raises_on_invalid_json PASSED
```

Notice what these tests do *not* test: they don't test whether the
LLM produces the right classification for a given question. That's
not the agent's job — the agent's job is to ask the LLM, parse the
response, and return a typed result. Whether the *LLM* gets the
classification right is verified by benchmarks (Module 9) against a
real model.

This separation is the whole point of decomposition: the agent's
plumbing is unit-testable; the agent's quality is benchmark-tested.
Don't conflate them.

### Checkpoint 4 — The SQL generation prompt

Create `agent/engine/prompt_templates/sql_generation.txt`:

```
You are a SQL expert. Generate a single SQL query that answers the
user's question.

Return ONLY the SQL. No markdown fences. No explanation. No prose.

If the question genuinely cannot be answered with the available
data, respond with exactly:
CANNOT_ANSWER: <one-sentence reason>

## Available tables (use only these)
{table_schemas}

## Examples
{examples}

## Question
{question}

Generate the SQL now.
```

This is much simpler than Tiri's `sql_generation.txt` (which has
~12 sections). Start narrow; you can grow later.

`{table_schemas}` and `{examples}` are placeholders the SQL agent
will fill from the `ContextPackage`. Helper functions for the
formatting:

In `agent/engine/agents/sql_agent.py`:

```python
"""SQLAgent — generates and validates SQL for a question."""
from __future__ import annotations
import logging
import re

from agent.data_models import (
    ContextPackage, IntentResult, SQLResult, LLMMessage,
)
from agent.providers.base import LLMProvider, QueryProvider, LLMProviderError
from agent.engine.prompts import load_template, render


_TEMPLATE = load_template("sql_generation.txt")
_log = logging.getLogger("agent.sql")
_CANNOT_ANSWER_PREFIX = "CANNOT_ANSWER:"


class SQLAgent:
    def __init__(
        self,
        llm: LLMProvider,
        query: QueryProvider,
        max_retries: int = 3,
    ) -> None:
        self._llm = llm
        self._query = query
        self._max_retries = max_retries

    async def run(
        self,
        question: str,
        context: ContextPackage,
        intent: IntentResult | None = None,
    ) -> SQLResult:
        relevant = intent.relevant_tables if intent and intent.relevant_tables else list(context.tables)

        system_prompt = render(
            _TEMPLATE,
            table_schemas=_format_schemas(relevant, context.tables),
            examples=_format_examples(context),
            question=question,
        )

        messages = [LLMMessage(role="system", content=system_prompt)]

        attempt = 0
        last_error: str | None = None
        while attempt < self._max_retries:
            attempt += 1
            try:
                response = await self._llm.complete(messages, task="sql")
            except LLMProviderError as e:
                return SQLResult(
                    is_valid=False, attempts=attempt,
                    error=f"LLM call failed: {e}",
                )

            candidate = _strip_markdown_fences(response.content.strip())

            if candidate.startswith(_CANNOT_ANSWER_PREFIX):
                reason = candidate[len(_CANNOT_ANSWER_PREFIX):].strip()
                return SQLResult(
                    is_valid=False, attempts=attempt,
                    error=f"CANNOT_ANSWER: {reason}",
                )

            result = await self._query.validate(candidate)
            if result.is_valid:
                return SQLResult(
                    is_valid=True, attempts=attempt, sql=candidate,
                )

            last_error = result.error or "validation failed"
            _log.info(
                "SQLAgent attempt %d invalid: %s", attempt, last_error
            )
            # Feed the validator's error back in for the next attempt.
            messages.append(LLMMessage(role="assistant", content=candidate))
            messages.append(LLMMessage(
                role="user",
                content=(
                    f"That SQL has an error: {last_error}\n"
                    f"Please fix it and return ONLY the corrected SQL."
                ),
            ))

        return SQLResult(
            is_valid=False, attempts=attempt,
            error=f"exhausted {self._max_retries} attempts; last: {last_error}",
        )


_FENCE = re.compile(r"^```(?:sql)?\s*|\s*```\s*$", re.IGNORECASE)


def _strip_markdown_fences(s: str) -> str:
    s = _FENCE.sub("", s).strip()
    return s


def _format_schemas(table_names, all_tables) -> str:
    lines = []
    for name in table_names:
        meta = all_tables.get(name)
        if not meta:
            continue
        lines.append(f"-- {name}: {meta.description or '(no description)'}")
        for c in meta.columns:
            lines.append(f"--   {c.name} ({c.data_type})")
    return "\n".join(lines) if lines else "(no tables)"


def _format_examples(ctx) -> str:
    if not ctx.retrieved_examples:
        return "(none)"
    chunks = []
    for e in ctx.retrieved_examples[:3]:
        chunks.append(f"-- Q: {e.question}\n{e.sql}")
    return "\n\n".join(chunks)
```

### Checkpoint 5 — Test SQLAgent: happy path and retry path

`tests/test_sql_agent.py`:

```python
import pytest
from agent.data_models import (
    ContextPackage, TableMeta, ColumnMeta, IntentResult,
    LLMResponse, ValidationResult,
)
from agent.engine.agents.sql_agent import SQLAgent
from agent.providers.base import LLMProvider, QueryProvider, LLMProviderError


class FakeLLM(LLMProvider):
    """Returns queued responses in order."""
    def __init__(self, queued: list[str]) -> None:
        self._queue = list(queued)
        self.call_count = 0

    async def complete(self, messages, **kwargs):
        self.call_count += 1
        if not self._queue:
            raise AssertionError("FakeLLM exhausted")
        return LLMResponse(content=self._queue.pop(0))

    async def stream(self, *a, **kw): raise NotImplementedError
    async def embed(self, *a, **kw): raise NotImplementedError


class FakeQuery(QueryProvider):
    """Returns queued ValidationResults in order."""
    def __init__(self, queued: list[ValidationResult]) -> None:
        self._queue = list(queued)

    async def validate(self, sql, **kwargs):
        if not self._queue:
            raise AssertionError("FakeQuery exhausted")
        return self._queue.pop(0)

    async def execute(self, *a, **kw): raise NotImplementedError


def _ctx() -> ContextPackage:
    return ContextPackage(
        question="",
        tables={
            "main.orders": TableMeta(
                full_name="main.orders",
                description="orders",
                columns=[
                    ColumnMeta(name="amount", data_type="DECIMAL"),
                    ColumnMeta(name="region", data_type="VARCHAR"),
                ],
            )
        },
        retrieved_examples=[],
        schema_descriptions={},
    )


@pytest.mark.asyncio
async def test_sql_happy_path():
    llm = FakeLLM(["SELECT region, SUM(amount) FROM orders GROUP BY region"])
    q = FakeQuery([ValidationResult(is_valid=True)])
    agent = SQLAgent(llm, q)
    result = await agent.run("revenue by region", _ctx())
    assert result.is_valid
    assert result.attempts == 1
    assert "SUM(amount)" in result.sql


@pytest.mark.asyncio
async def test_sql_retries_on_validation_error():
    llm = FakeLLM([
        "SELECT bad_column FROM orders",          # attempt 1: invalid
        "SELECT region FROM orders",              # attempt 2: valid
    ])
    q = FakeQuery([
        ValidationResult(is_valid=False, error="column bad_column does not exist"),
        ValidationResult(is_valid=True),
    ])
    agent = SQLAgent(llm, q, max_retries=3)
    result = await agent.run("revenue by region", _ctx())
    assert result.is_valid
    assert result.attempts == 2
    assert llm.call_count == 2


@pytest.mark.asyncio
async def test_sql_strips_markdown_fences():
    llm = FakeLLM([
        "```sql\nSELECT region FROM orders\n```",
    ])
    q = FakeQuery([ValidationResult(is_valid=True)])
    agent = SQLAgent(llm, q)
    result = await agent.run("regions", _ctx())
    assert result.is_valid
    assert result.sql == "SELECT region FROM orders"


@pytest.mark.asyncio
async def test_sql_returns_cannot_answer():
    llm = FakeLLM(["CANNOT_ANSWER: no churn data in this room"])
    q = FakeQuery([])  # should not be called
    agent = SQLAgent(llm, q)
    result = await agent.run("why are customers churning?", _ctx())
    assert not result.is_valid
    assert "CANNOT_ANSWER" in result.error
    assert "churn data" in result.error


@pytest.mark.asyncio
async def test_sql_exhausts_retries_and_returns_error():
    llm = FakeLLM([
        "SELECT x FROM orders",
        "SELECT y FROM orders",
        "SELECT z FROM orders",
    ])
    q = FakeQuery([
        ValidationResult(is_valid=False, error="x missing"),
        ValidationResult(is_valid=False, error="y missing"),
        ValidationResult(is_valid=False, error="z missing"),
    ])
    agent = SQLAgent(llm, q, max_retries=3)
    result = await agent.run("nope", _ctx())
    assert not result.is_valid
    assert result.attempts == 3
    assert "exhausted" in result.error
```

Run:

```bash
pytest tests/test_sql_agent.py -v
```

Expected output (all five passing).

The third test (`test_sql_strips_markdown_fences`) is the most
production-relevant of the bunch. Real models — especially smaller
ones — wrap their output in ```` ```sql ... ``` ```` despite the
prompt forbidding it. The strip step is what makes the agent robust
to that.

### Checkpoint 6 — Chain them in a test

This isn't the engine (that's Module 7), but it proves the pieces
connect. Add to `tests/test_intent_agent.py`:

```python
@pytest.mark.asyncio
async def test_intent_then_sql_in_sequence():
    """Smoke test: IntentAgent → SQLAgent chain works on fixtures."""
    from agent.engine.agents.sql_agent import SQLAgent
    import json

    intent_llm = FakeLLM(json.dumps({
        "intent": "answerable",
        "reasoning": "revenue is in orders",
        "relevant_tables": ["main.orders"],
    }))
    intent = await IntentAgent(intent_llm).run("revenue by region", _ctx())
    assert intent.kind == "answerable"

    # Now hand off to SQL.
    from tests.test_sql_agent import FakeLLM as SqlFake, FakeQuery, _ctx as _sql_ctx
    from agent.data_models import ValidationResult
    sql_llm = SqlFake([
        "SELECT region, SUM(amount) FROM orders GROUP BY region",
    ])
    sql_q = FakeQuery([ValidationResult(is_valid=True)])
    sql_result = await SQLAgent(sql_llm, sql_q).run(
        "revenue by region", _sql_ctx(), intent
    )
    assert sql_result.is_valid
    assert "SUM(amount)" in sql_result.sql
```

This is the simplest possible "two agents in a row" test. Module 7
turns this kind of thing into a real engine. For now it proves your
pieces fit together.

### Checkpoint 7 — The end-to-end check (with a real model)

Optional but recommended: run your agents against the actual LLM,
not just fakes.

Create `scripts/try_intent.py`:

```python
"""Hit the real model with a few questions. Manual smoke check."""
import asyncio
import os
from agent.data_models import ContextPackage, TableMeta
from agent.engine.agents.intent_agent import IntentAgent
from agent.providers.openai_llm import OpenAILLMProvider


async def main():
    llm = OpenAILLMProvider(api_key=os.environ["OPENAI_API_KEY"])
    agent = IntentAgent(llm)

    ctx = ContextPackage(
        question="",
        tables={
            "main.orders": TableMeta(
                full_name="main.orders",
                description="Customer orders, one row per order.",
            )
        },
        retrieved_examples=[],
        schema_descriptions={},
    )

    questions = [
        "What was last quarter's revenue by region?",
        "Why is revenue declining?",
        "tell me about orders",                # ambiguous
    ]
    for q in questions:
        result = await agent.run(q, ctx)
        print(f"\nQ: {q}")
        print(f"   kind={result.kind}")
        print(f"   reasoning={result.reasoning}")
        if result.clarification:
            print(f"   clarification={result.clarification}")
        if result.refusal_reason:
            print(f"   refusal_reason={result.refusal_reason}")


asyncio.run(main())
```

Run:

```bash
python scripts/try_intent.py
```

You should see (with some variation):

- "revenue by region" → `kind=answerable`
- "why is revenue declining?" → `kind=refused` (or `clarification_needed`,
  depending on your prompt)
- "tell me about orders" → `kind=clarification_needed`

If the classification is wrong, the fix is almost always in the
prompt template, not in the Python code. Edit
`intent_classification.txt`, re-run. This is the inner loop you'll
spend most of Module 9 in.

### Definition of done

- `agent/engine/agents/intent_agent.py` and `sql_agent.py` exist and
  pass their tests.
- `agent/engine/prompt_templates/intent_classification.txt` and
  `sql_generation.txt` exist and render without errors.
- The chained test passes.
- (Optional) The real-model smoke test produces reasonable
  classifications for at least three test questions.
- Commit:
  ```bash
  git add . && git commit -m "module-06: IntentAgent + SQLAgent with retry loop"
  ```

### Common pitfalls

1. **The fake LLM returns the same content forever.** Tests pass for
   the wrong reason — the second attempt happens to match the first.
   Use a queue of responses (`FakeLLM([resp1, resp2, ...])`) so each
   call returns the next one.

2. **You try to repair invalid JSON.** Tempting. "If the JSON is
   missing a brace, just add it." This is how silent data corruption
   happens. Either the model produced parseable JSON or it didn't.
   If it didn't, raise.

3. **Your retry loop doesn't append the error to the messages.** The
   model gets three identical attempts at the same question and
   produces the same wrong answer three times. The whole point of
   retry is to *show the model what went wrong*. Without that, the
   loop is wasted calls.

4. **You catch broad `Exception` in the agent.** A bug in your
   parsing code now looks like "LLM failed." Catch
   `LLMProviderError` (from your provider layer) and let everything
   else propagate.

5. **You test the agent by checking the SQL it generates against a
   regex.** Tempting. Don't. The agent's job is to pass `validate()`
   and return a typed result; *what* SQL the model produces is the
   model's responsibility, verified at the benchmark level (Module
   9). Testing for SQL substrings in unit tests is brittle and slows
   prompt iteration.

6. **The `_FENCE` regex eats real backticks in the SQL.** The
   pattern here only strips fences at the start and end of the
   string. If your SQL legitimately contains backticks (rare but
   possible), make sure your regex anchors at `^` and `$`.

## Stretch

1. **Add a `ClarifyAgent`.** It takes a question that `IntentAgent`
   classified as `clarification_needed` and generates a short,
   user-facing clarifying question. Test the same way: fake LLM,
   fixture context, assert the output.

2. **Add a regex scan to a `SynthesisAgent`-style output.** Even
   without building the full synthesis agent, write a helper that
   takes generated text and asserts no banned causal phrases appear.
   Test it.

3. **Wire structured-output (JSON mode).** OpenAI's
   `response_format={"type": "json_object"}` makes the model return
   valid JSON. Modify your `OpenAILLMProvider.complete()` to accept
   a `response_format` argument and have `IntentAgent` use it. The
   defensive JSON parser becomes near-redundant but still catches
   models that don't support the flag.

## Reflection

1. The `IntentAgent` and `SQLAgent` are independent. They could be
   tested separately, deployed separately, scaled separately, and
   even served by different models. In what kind of project would
   you collapse them into a single agent anyway? What would you give
   up?

2. The retry loop in `SQLAgent` has `max_retries=3`. What's the
   right number for *your* application? What signals would tell you
   to raise it or lower it? (Hint: cost per attempt, the marginal
   value of the third attempt vs the first.)

3. Tiri uses `task="sql"` and `task="intent"` parameters on
   `llm.complete()` even when there's only one backend. Why? What
   capability does this leave on the table for Module 10?

4. Your `IntentAgent` has no way to express "the user is asking
   *two* questions at once." How would you extend it to handle that
   case — a different intent kind, a list of intents, a separate
   agent? Sketch the API; don't implement.
