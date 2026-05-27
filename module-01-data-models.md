# Module 1 — Data models as contracts

> **The rule:** Foundation before fancy. Data models are contracts.
> Invariants belong in `__post_init__`, not in the call site that
> hopes the data is right.

In this module you'll build the dataclasses that every later module
will pass around. Boring. Necessary. The discipline you bring here
determines how easy or impossible Module 6 is.

## Read

1. **`tiri/docs/data_models.md`** — the whole thing. Pay attention to:
   - The "Test cases" table near the end. Every row is a requirement,
     including the negative ones ("X MUST raise ValueError").
   - The mutual-exclusion rule on `ConversationTurn`. This is the most
     famous example of "invariant lives in the data model, not the
     code that constructs it" and the easiest one to get wrong.

2. **`tiri/tiri/data_models.py`** — skim. Don't read every dataclass.
   Spend your reading time on:
   - `ConversationTurn` (around line 583) — note `__post_init__`.
   - `ContextPackage` (around line 624) — note how many fields it has
     and how little logic. The simplicity is the point.
   - `RoomConfig` (around line 238) — the largest dataclass in the
     system. Its `from_dict` and `to_dict` methods are worth a look,
     but again, no logic beyond validation.

## Concepts

### Why dataclasses, not dicts

Three reasons, in order of importance:

1. **Invariants have somewhere to live.** A dict has no opinion about
   whether `sql` and `clarification_question` are both set. A
   dataclass with `__post_init__` does. The invariant is enforced
   exactly once, at construction. Every downstream consumer can trust
   it.

2. **Types are documentation that doesn't lie.** A reader of
   `def chat(turn: ConversationTurn)` knows what shape `turn` is
   without running anything. A reader of `def chat(turn: dict)` does
   not.

3. **Mistakes happen at the boundary, not the middle.** When you build
   the dataclass *at the boundary* — when reading a config file, when
   accepting an API request — the rest of the system never sees an
   invalid object. Bugs surface where the data enters, not three
   layers deep where you can't find them.

### Why Python's `@dataclass`, not Pydantic

Pydantic is excellent for parsing untrusted input. Tiri uses plain
`@dataclass` for the internal model because:

- The internal data is trusted — it's already been validated at the
  API boundary (where you would use Pydantic).
- Plain dataclasses have zero dependency footprint and start fast.
- `__post_init__` gives you the validation you actually need at the
  internal layer (invariants, not parsing).

This is a judgment call, not a rule. If your project already uses
Pydantic everywhere, use Pydantic everywhere. The principle — the
invariant lives with the data — is what matters. The library choice
is secondary.

### Mutual exclusion as a worked example

`ConversationTurn` MUST have exactly one of `{sql,
clarification_question, error}` set. The reference implementation:

```python
def __post_init__(self) -> None:
    set_count = sum(
        1
        for v in (self.sql, self.clarification_question, self.error)
        if v is not None
    )
    if set_count != 1:
        raise ValueError(
            "ConversationTurn MUST have exactly one of "
            "{sql, clarification_question, error} set "
            f"(got {set_count}); these are mutually exclusive."
        )
```

Look at what this does and does not do:

- Does: raise on construction if the invariant is violated.
- Does not: try to "fix" the input by picking one and discarding the
  others. Silent repair is how trust-eroding bugs get into production.
- Does: include the actual `got` count in the error message. Future
  you, debugging this at 2am, will thank present you.

### Tested by spec, not by hope

Open `tiri/docs/data_models.md` and find the "Test cases" table. Every
row is the contract this dataclass owes the rest of the system. Every
row gets a test. There is no row called "make sure it's roughly
right." If the doc says MUST, the test asserts it. If the doc says
SHOULD, the test warns on failure.

This discipline is unusual. It is also why a 467-test suite runs in
~3 seconds and the system holds together across 12 build steps.

## Lab

You're going to build a slim parallel of Tiri's data model layer for
your own agent — the one whose refusal manifesto you wrote in
Module 0. Six checkpoints. Don't proceed past one until the previous
is green.

### Checkpoint 1 — Project scaffold

In `~/my-agent/`:

```bash
mkdir -p agent tests
touch agent/__init__.py agent/data_models.py
touch tests/__init__.py tests/test_data_models.py
```

Create `pyproject.toml`:

```toml
[project]
name = "my-agent"
version = "0.0.1"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=7"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Install:

```bash
pip install -e ".[dev]"
pytest tests/ -v
```

Expected output:

```
============== no tests ran in 0.01s ===============
```

That's correct — you have no tests yet. The scaffold is working.

### Checkpoint 2 — The `Question` and `Evidence` dataclasses

In `agent/data_models.py`, paste this starting skeleton and fill in
the `TODO` markers:

```python
"""Data models — contracts every later layer relies on."""
from __future__ import annotations
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Literal


@dataclass
class Question:
    """The user's input to the agent.

    Constructed at the API boundary, passed through the pipeline.
    """
    question_id: str
    text: str
    asked_at: datetime
    # TODO: add ONE domain-specific field that came out of your
    # manifesto. Examples:
    #   user_role: str = ""           # for an internal agent
    #   audience: Literal["public", "internal"] = "internal"
    #   urgency: Literal["high", "normal"] = "normal"
    # Only add what your manifesto's audience actually needs.


@dataclass
class Evidence:
    """One supporting fact the agent surfaces with its answer.

    Each piece of evidence has a source the user can verify and a
    confidence the user can weigh.
    """
    source: str            # human-readable: "orders table", "Q3 board memo"
    excerpt: str           # the actual data: "12,453 orders in Q3"
    confidence: Literal["high", "medium", "low"]

    def __post_init__(self) -> None:
        if self.confidence not in ("high", "medium", "low"):
            raise ValueError(
                f"confidence must be high/medium/low, got "
                f"{self.confidence!r}"
            )
```

Why a `__post_init__` check when `Literal` already restricts the
type? Because `Literal` is a *type-checker* hint, not a *runtime*
constraint. A `dict.get()` or a JSON deserializer can hand you
`"HIGH"` or `"unknown"` and Python won't complain. The runtime check
catches the bug at construction, where you can see the bad value.

### Checkpoint 3 — Tests for `Question` and `Evidence`

In `tests/test_data_models.py`:

```python
from datetime import datetime, timezone
import pytest

from agent.data_models import Question, Evidence


def test_question_constructs():
    q = Question(
        question_id="q-1",
        text="What was Q3 revenue by region?",
        asked_at=datetime(2026, 1, 15, tzinfo=timezone.utc),
    )
    assert q.text.startswith("What")


def test_evidence_accepts_valid_confidence():
    for conf in ("high", "medium", "low"):
        e = Evidence(source="orders", excerpt="12k orders", confidence=conf)
        assert e.confidence == conf


def test_evidence_rejects_invalid_confidence():
    with pytest.raises(ValueError, match="high/medium/low"):
        Evidence(source="orders", excerpt="12k", confidence="HIGH")
```

Run:

```bash
pytest tests/ -v
```

Expected output:

```
tests/test_data_models.py::test_question_constructs PASSED
tests/test_data_models.py::test_evidence_accepts_valid_confidence PASSED
tests/test_data_models.py::test_evidence_rejects_invalid_confidence PASSED
```

If the third test fails (no `ValueError` raised), your
`__post_init__` isn't running — check that you didn't forget the
`@dataclass` decorator or the `from __future__ import annotations`
line.

### Checkpoint 4 — The `Answer` dataclass

The synthesized response the agent sends back. Add to
`agent/data_models.py`:

```python
@dataclass
class Answer:
    """The agent's synthesized response.

    `caveats` is where the agent puts honest limitations — "this
    excludes orders from region X because data wasn't available",
    "this assumes the fiscal calendar starts in February", etc.
    Empty caveats list is suspicious in a real system; an answer with
    zero qualifications is almost always overstated.
    """
    text: str
    evidence: list[Evidence] = field(default_factory=list)
    caveats: list[str] = field(default_factory=list)
```

Add a test:

```python
from agent.data_models import Answer


def test_answer_round_trip_through_dict():
    a = Answer(
        text="Q3 revenue was $1.2M, up 14% over Q2.",
        evidence=[
            Evidence(source="orders", excerpt="$1.2M sum", confidence="high"),
            Evidence(source="orders_q2", excerpt="$1.05M sum", confidence="high"),
        ],
        caveats=["Excludes returns and refunds.", "Fiscal Q3 = Aug–Oct."],
    )
    d = asdict(a)
    assert d["text"] == a.text
    assert len(d["evidence"]) == 2
    assert d["evidence"][0]["confidence"] == "high"
```

Don't forget to add `asdict` to your imports in the test file.

### Checkpoint 5 — The `Turn` dataclass (the invariant lives here)

This is the one with the mutual-exclusion invariant.

```python
@dataclass
class Turn:
    """One complete exchange — question in, exactly one of three out.

    A Turn must hold exactly one of:
      - `answer`: the agent successfully answered.
      - `unable_to_answer`: the agent honestly declined (data
        insufficient, scope, conflicting sources). The string is the
        user-facing explanation.
      - `clarification_question`: the agent needs more from the user
        before it can proceed.

    The mutual exclusion is enforced at construction so no downstream
    consumer ever has to ask "which one is set?" with a wrong answer
    in mind.
    """
    turn_id: str
    question: Question
    answer: Answer | None = None
    unable_to_answer: str | None = None
    clarification_question: str | None = None

    def __post_init__(self) -> None:
        set_count = sum(
            1
            for v in (
                self.answer,
                self.unable_to_answer,
                self.clarification_question,
            )
            if v is not None
        )
        if set_count != 1:
            raise ValueError(
                "Turn MUST have exactly one of "
                "{answer, unable_to_answer, clarification_question} "
                f"set (got {set_count})."
            )
```

Note the error message includes the `set_count`. Future-you debugging
this at 2am will see "got 0" or "got 2" and know exactly what shape
of bad input was passed.

### Checkpoint 6 — Tests for `Turn` (including the negative cases)

```python
from agent.data_models import Turn


def _q():
    """Test-only Question factory."""
    return Question(
        question_id="q-1",
        text="What was Q3 revenue?",
        asked_at=datetime(2026, 1, 15, tzinfo=timezone.utc),
    )


def test_turn_with_answer():
    t = Turn(
        turn_id="t-1",
        question=_q(),
        answer=Answer(text="$1.2M"),
    )
    assert t.answer.text == "$1.2M"


def test_turn_with_unable_to_answer():
    t = Turn(
        turn_id="t-1",
        question=_q(),
        unable_to_answer="No Q3 data available for this region.",
    )
    assert t.unable_to_answer.startswith("No Q3")


def test_turn_with_clarification():
    t = Turn(
        turn_id="t-1",
        question=_q(),
        clarification_question="Which region did you mean — North or APAC?",
    )
    assert "North" in t.clarification_question


def test_turn_raises_when_zero_set():
    with pytest.raises(ValueError, match="got 0"):
        Turn(turn_id="t-1", question=_q())


def test_turn_raises_when_two_set():
    with pytest.raises(ValueError, match="got 2"):
        Turn(
            turn_id="t-1",
            question=_q(),
            answer=Answer(text="$1.2M"),
            unable_to_answer="No data.",
        )


def test_turn_raises_when_all_three_set():
    with pytest.raises(ValueError, match="got 3"):
        Turn(
            turn_id="t-1",
            question=_q(),
            answer=Answer(text="$1.2M"),
            unable_to_answer="No data.",
            clarification_question="Which region?",
        )
```

Run:

```bash
pytest tests/ -v
```

Expected output:

```
tests/test_data_models.py::test_question_constructs PASSED
tests/test_data_models.py::test_evidence_accepts_valid_confidence PASSED
tests/test_data_models.py::test_evidence_rejects_invalid_confidence PASSED
tests/test_data_models.py::test_answer_round_trip_through_dict PASSED
tests/test_data_models.py::test_turn_with_answer PASSED
tests/test_data_models.py::test_turn_with_unable_to_answer PASSED
tests/test_data_models.py::test_turn_with_clarification PASSED
tests/test_data_models.py::test_turn_raises_when_zero_set PASSED
tests/test_data_models.py::test_turn_raises_when_two_set PASSED
tests/test_data_models.py::test_turn_raises_when_all_three_set PASSED

========== 10 passed in 0.04s ==========
```

### Verify the invariant is doing real work

This is a sanity check. Temporarily comment out the body of
`Turn.__post_init__`:

```python
def __post_init__(self) -> None:
    pass  # ← temporarily
```

Run the tests:

```bash
pytest tests/ -v
```

The three "raises" tests should now *fail*:

```
tests/test_data_models.py::test_turn_raises_when_zero_set FAILED
tests/test_data_models.py::test_turn_raises_when_two_set FAILED
tests/test_data_models.py::test_turn_raises_when_all_three_set FAILED
```

This proves the invariant is the thing your tests are testing — not
some accidental side-effect. **Restore the original `__post_init__`
body and re-run** to get back to green.

### Definition of done

- `agent/data_models.py` defines `Question`, `Evidence`, `Answer`,
  `Turn`.
- `tests/test_data_models.py` has at least the 10 tests above and
  all pass.
- The "comment out `__post_init__`" sanity check demonstrates the
  invariant is the actual thing being tested.
- Commit:
  ```bash
  git add . && git commit -m "module-01: data models with invariants"
  ```

### Common pitfalls

1. **`from __future__ import annotations` missing.** Without it,
   `list[Evidence]` and `Answer | None` annotations fail on
   Python 3.10. Use Python 3.11+ or keep that import.

2. **`__post_init__` doesn't run on `dataclasses.replace`.** If you
   create a `Turn` and then call `dataclasses.replace(turn,
   answer=None)` to clear the answer, the result is *not*
   re-validated. If you need to mutate, construct a new instance via
   the normal constructor.

3. **Tests pass for the wrong reason.** A `Turn` test that asserts
   `with pytest.raises(TypeError)` instead of `ValueError` will
   "pass" if you forgot the `@dataclass` decorator entirely (which
   would raise `TypeError` on construction). Use `pytest.raises(ValueError, match="...")`
   with a substring match against the message you wrote — this
   catches that bug.

4. **You add `frozen=True` and then can't build a `Turn` step by
   step.** `frozen=True` makes mutation impossible after construction,
   which is great — but means you must pass everything to the
   constructor at once. For Module 1 leave dataclasses mutable; we'll
   discuss `frozen=True` in the reflection.

## Stretch

Add a `ContextPackage`-equivalent for your domain. In Tiri it bundles
everything an agent needs before any LLM call: table schemas, joins,
SQL snippets, instructions, retrieved examples, conversation history.
What does your equivalent bundle? Build it. The test is whether you
can populate every field from a fixture and assert the result is
stable across reconstruction.

Bonus: write a `from_dict` classmethod that round-trips through JSON.
Tiri's `RoomConfig.from_dict` is the reference.

## Reflection

1. You added a `__post_init__` validation to `Evidence.confidence`.
   What other field in your model has an invariant that could go in
   `__post_init__` and currently doesn't? Add it, or write down why
   you decided not to.

2. The mutual exclusion on `Turn` is enforced at construction. What's
   the equivalent of "construction" at your system's API boundary —
   the moment when external input becomes an internal object? Will
   that moment always go through your dataclass constructor, or are
   there back doors (a `from_dict`, a deserializer, a `replace()`)
   where the invariant could be bypassed? If yes, decide whether to
   close them now or accept the risk.

3. If a future maintainer added a fourth mutually-exclusive field to
   `Turn` (say, `tool_call: str | None`), what would they need to
   change, and how would they know to change it? Would your tests
   catch the mistake of forgetting? If not, what's the test that
   would?
