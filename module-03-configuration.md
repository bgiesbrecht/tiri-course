# Module 3 — Configuration & the container

> **The rule:** Configuration is a system, not a dict. One wiring
> point. TOML plus environment variables. Secrets never in code.

By the end of this module your agent will be configurable from a file
*and* environment variables, will swap LLM backends without a code
change, and will have exactly one place where providers get
instantiated. That last property is what makes Module 7's
orchestration tractable.

This module is the most "plumbing" of the course. It is also the one
that pays off the most over time. The systems that get configuration
wrong are the systems that grow a `config.py` with 47 globals and a
`utils.py` that imports everything. Don't be that system.

## Read

1. **`tiri/docs/configuration.md`** — the whole thing.

2. **`tiri/tiri/config.py`** — read top to bottom. Yes, all 500 lines.
   The reading order:
   - The `ConfigurationError`, `ProviderBackendConfig`,
     `RoutingConfig`, and `Config` dataclasses (lines 26–120).
   - `Config.load()` (around line 115). Trace what it reads from
     `tiri.toml`, what it reads from env, and what it does when
     neither is present.
   - The helpers `_env_or`, `_env_for_provider_api_key`,
     `_default_completion_model`, `_substitute_env_vars`. These are
     where the "TOML can reference `${VAR}` and it gets expanded"
     behavior lives.

3. **`tiri/tiri/container.py`** — read top to bottom.
   - The `RouterLLMProvider` class — note that it is the *only* type
     the container ever returns for LLM. Single-backend installations
     get a `RouterLLMProvider` that routes every task to the same
     backend. Multi-backend (Module 10) gets the same class with a
     richer routing map. **One LLM provider type to rule them all.**
   - `build_container()` and the `_build_*` helpers. Each builds one
     concrete provider from `Config`. The function is large but
     boring — that's the goal.

4. **`tiri/tiri.toml.example` and `tiri/.env.example`** — five-minute
   read each. These are the surface area a user of the system sees.

## Concepts

### TOML for shape, env for secrets

The rule, in one line: **anything you'd be embarrassed to paste in a
GitHub issue lives in env vars; everything else lives in TOML.**

A `tiri.toml`:

```toml
[llm.backends.openai_main]
type = "openai"
model = "gpt-4o-mini"
api_key = "${OPENAI_API_KEY}"  # ← string, expanded at load time

[llm.routing]
intent = "openai_main"
sql = "openai_main"
synthesis = "openai_main"
embed = "openai_main"

[query]
provider = "duckdb"
data_dir = "./data"
```

A `.env.local`:

```bash
OPENAI_API_KEY=sk-...
```

The TOML is committable. The `.env.local` is gitignored. The
substitution happens once, at load time, inside `Config.load()`. No
other code reads env vars directly — if a future maintainer wants to
add a new setting, they add it to the dataclass and the loader, not
sprinkle `os.environ.get(...)` across the codebase.

This is why `Config` is a single dataclass with ~20 fields, not a
dict you query. The dataclass is the schema. If you can't fit your
setting into the dataclass, you should think about whether it really
belongs in config or whether it belongs somewhere else (a database, a
provider's internal state, a CLI flag).

### The container pattern, simplified

"Dependency injection" in the Java sense is over-engineered for
Python. Tiri's pattern is simpler than it sounds:

```python
def build_container(cfg: Config) -> dict[str, Any]:
    llm = _build_llm(cfg)
    catalog = _build_catalog(cfg)
    query = _build_query(cfg)
    # ... etc.
    return {"llm": llm, "catalog": catalog, "query": query, ...}
```

That's the whole pattern. One function, takes `Config`, returns a
dict of providers. Every consumer (agents, the engine, the API)
receives providers from this dict. Nobody calls a provider
constructor directly outside of `container.py`.

Why this is enough:

- Tests build a test container with mocked providers.
- Production builds a real container with real providers.
- Both paths use the same engine code.
- New providers add one `_build_*` helper. No framework to learn.

If your project grows and you want a richer DI system later, you can
swap to one — but you almost certainly won't need to.

### Lazy imports for optional dependencies

Your agent might support OpenAI, Anthropic, *and* Databricks LLM
backends. A user with only an OpenAI key shouldn't need to install
the `anthropic` or `databricks` SDKs.

The trick: import the SDK *inside* the `_build_llm` branch, not at
the top of `container.py`.

```python
def _instantiate_llm(bc: ProviderBackendConfig) -> LLMProvider:
    if bc.type == "openai":
        from agent.providers.openai_llm import OpenAILLMProvider
        return OpenAILLMProvider(api_key=bc.api_key, model=bc.model)
    if bc.type == "anthropic":
        from agent.providers.anthropic_llm import AnthropicLLMProvider
        return AnthropicLLMProvider(...)
    raise ConfigurationError(f"unknown llm backend type: {bc.type}")
```

Now installing `agent` doesn't pull `anthropic` unless your config
asks for it. This pattern matters more as your provider count grows.

### The "RouterLLMProvider always" rule

Tiri's container always returns a `RouterLLMProvider`, even when only
one backend is configured. The single-backend version routes every
task to that backend and is functionally identical to wrapping the
backend directly.

Why bother? Because Module 10 adds multi-model routing as a wiring
change — the engine code already calls `llm.complete(task="sql",
...)` and the router decides where it goes. If the engine had been
written against a raw backend type, Module 10 would require a refactor
of every agent. By committing to the router from day one, the cost is
~30 lines now and zero refactoring later.

Pay the small cost now. It is one of the highest-ROI decisions in the
whole architecture.

## Lab

### Checkpoint 1 — `Config` and `Config.load`

Create `agent/config.py`. Start from this skeleton:

```python
"""Runtime configuration for the agent."""
from __future__ import annotations
from dataclasses import dataclass, field
import os
import tomllib  # Python 3.11+
from pathlib import Path
from typing import Any


class ConfigurationError(Exception):
    """Raised when configuration is invalid or missing required values."""


@dataclass
class LLMBackendConfig:
    type: str          # "openai" | "anthropic"
    model: str
    api_key: str       # ${ENV_VAR} expanded at load time


@dataclass
class Config:
    llm_backend: LLMBackendConfig
    query_data_dir: str = "./data"
    vector_db_path: str = ":memory:"
    chroma_collection: str = "examples"

    @classmethod
    def load(cls, toml_path: str = "agent.toml") -> "Config":
        raw = _read_toml(toml_path)
        if raw is None:
            return cls._from_env()
        return cls._from_toml(raw)

    @classmethod
    def _from_env(cls) -> "Config":
        # TODO: synthesize a single-backend Config from env vars.
        # Look at OPENAI_API_KEY / ANTHROPIC_API_KEY to decide which.
        ...

    @classmethod
    def _from_toml(cls, raw: dict[str, Any]) -> "Config":
        # TODO: read raw['llm']['backend'], expand ${VAR} references,
        # build LLMBackendConfig.
        ...


def _read_toml(path: str) -> dict | None:
    p = Path(path)
    if not p.exists():
        return None
    with p.open("rb") as f:
        return tomllib.load(f)


def _expand_env(value: Any) -> Any:
    """Replace ${VAR} in strings; recurse into dicts/lists."""
    # TODO: implement. See tiri/config.py:_substitute_env_vars for the reference.
    ...
```

Fill in the `TODO`s. Keep the surface small — one LLM backend, no
multi-model routing yet. You will widen this later, but the discipline
to start narrow is what keeps the file readable.

Then create your first config files:

`~/my-agent/agent.toml`:

```toml
[llm.backend]
type = "openai"
model = "gpt-4o-mini"
api_key = "${OPENAI_API_KEY}"
```

`~/my-agent/.env.local` (and add to `.gitignore` now):

```bash
OPENAI_API_KEY=sk-...
```

Test it:

```python
# tests/test_config.py
from agent.config import Config
import os


def test_config_loads_from_toml(tmp_path, monkeypatch):
    monkeypatch.setenv("OPENAI_API_KEY", "sk-test")
    toml = tmp_path / "agent.toml"
    toml.write_text(
        '[llm.backend]\n'
        'type = "openai"\n'
        'model = "gpt-4o-mini"\n'
        'api_key = "${OPENAI_API_KEY}"\n'
    )
    cfg = Config.load(str(toml))
    assert cfg.llm_backend.type == "openai"
    assert cfg.llm_backend.api_key == "sk-test"  # ← expanded


def test_config_falls_back_to_env_when_no_toml(monkeypatch):
    monkeypatch.setenv("OPENAI_API_KEY", "sk-test")
    cfg = Config.load("/nonexistent/path.toml")
    assert cfg.llm_backend.type == "openai"
    assert cfg.llm_backend.api_key == "sk-test"


def test_config_raises_when_missing_required(monkeypatch):
    monkeypatch.delenv("OPENAI_API_KEY", raising=False)
    monkeypatch.delenv("ANTHROPIC_API_KEY", raising=False)
    import pytest
    from agent.config import ConfigurationError
    with pytest.raises(ConfigurationError):
        Config.load("/nonexistent/path.toml")
```

Run these and make them pass before moving on:

```bash
pytest tests/test_config.py -v
```

Expected output:

```
tests/test_config.py::test_config_loads_from_toml PASSED
tests/test_config.py::test_config_falls_back_to_env_when_no_toml PASSED
tests/test_config.py::test_config_raises_when_missing_required PASSED
```

**Do not proceed to checkpoint 2 until the above pass.** Configuration
bugs caught here cost minutes; the same bugs caught after the
container is wired up cost hours.

### Checkpoint 2 — The container

Create `agent/container.py`. Skeleton:

```python
"""Wires Config → providers. The only place provider constructors are called."""
from __future__ import annotations
from typing import Any

from agent.config import Config, ConfigurationError
from agent.providers.base import LLMProvider, QueryProvider


def build_container(cfg: Config) -> dict[str, Any]:
    return {
        "llm": _build_llm(cfg),
        "query": _build_query(cfg),
        # vector + store come later
    }


def _build_llm(cfg: Config) -> LLMProvider:
    bc = cfg.llm_backend
    if bc.type == "openai":
        from agent.providers.openai_llm import OpenAILLMProvider
        return OpenAILLMProvider(api_key=bc.api_key, model=bc.model)
    if bc.type == "anthropic":
        from agent.providers.anthropic_llm import AnthropicLLMProvider
        return AnthropicLLMProvider(api_key=bc.api_key, model=bc.model)
    raise ConfigurationError(f"unknown llm backend type: {bc.type!r}")


def _build_query(cfg: Config) -> QueryProvider:
    from agent.providers.duckdb_query import DuckDBQueryProvider
    return DuckDBQueryProvider(data_dir=cfg.query_data_dir)
```

Note three things:

1. The SDK imports (`OpenAILLMProvider`, `DuckDBQueryProvider`) are
   *inside* the `_build_*` functions. This is the lazy-import pattern
   from the concepts section. A user with only an OpenAI key will
   never trigger `import anthropic`.
2. `_build_query` has no branch yet. When you add a second query
   backend (e.g., Databricks), it grows the same shape as `_build_llm`.
3. The return type is a plain `dict`. Not a class. Not a framework.
   A dict.

Test:

```python
# tests/test_container.py
from agent.config import Config, LLMBackendConfig
from agent.container import build_container
from agent.providers.base import LLMProvider, QueryProvider


def test_container_wires_providers_from_config():
    cfg = Config(
        llm_backend=LLMBackendConfig(
            type="openai", model="gpt-4o-mini", api_key="sk-test"
        ),
        query_data_dir="./data",
    )
    container = build_container(cfg)
    assert isinstance(container["llm"], LLMProvider)
    assert isinstance(container["query"], QueryProvider)


def test_container_raises_on_unknown_backend():
    import pytest
    from agent.config import ConfigurationError
    cfg = Config(
        llm_backend=LLMBackendConfig(
            type="badbackend", model="x", api_key="x"
        ),
    )
    with pytest.raises(ConfigurationError):
        build_container(cfg)
```

Run:

```bash
pytest tests/ -v
```

All tests from Modules 1, 2, and 3 should now pass together.

### Checkpoint 3 — Swap the backend (the proof)

This is the test that proves the architecture works. Add this test
*without changing any production code* outside of an `if bc.type ==
"anthropic"` branch in `_build_llm`:

```python
def test_container_swaps_llm_backend_via_config_only():
    """Same Config shape, different backend, no code changes."""
    cfg_openai = Config(
        llm_backend=LLMBackendConfig(
            type="openai", model="gpt-4o-mini", api_key="sk-fake"
        ),
    )
    cfg_anthropic = Config(
        llm_backend=LLMBackendConfig(
            type="anthropic", model="claude-haiku-4-5-20251001", api_key="sk-fake"
        ),
    )

    container_o = build_container(cfg_openai)
    container_a = build_container(cfg_anthropic)

    # Both implement LLMProvider; consumers don't know or care which.
    assert isinstance(container_o["llm"], LLMProvider)
    assert isinstance(container_a["llm"], LLMProvider)
    # Different concrete classes under the hood.
    assert type(container_o["llm"]) is not type(container_a["llm"])
```

Pass this test. You have now demonstrated, with an assertion, that
your config layer can swap backends without touching consumer code.
This is not a metaphor. It is the literal property the architecture
guarantees.

### Definition of done

- `agent/config.py` and `agent/container.py` exist, with the
  responsibilities described above.
- All tests pass: `pytest tests/ -v`.
- `agent.toml` exists, committed. `.env.local` exists, gitignored.
- `grep -r "os.environ" agent/` returns matches **only in
  `agent/config.py`**. Nowhere else.
- `grep -rn "import openai\|import anthropic" agent/` returns matches
  **only inside functions in `agent/container.py` and the corresponding
  files in `agent/providers/`**. Not at any module top level outside
  `providers/`.

Commit: `git commit -m "module-03: config + container with backend
swap"`.

### Common pitfalls

These come up almost every time. If you hit one, the bug is in the
list before it is in your code:

1. **`tomllib` not found.** It's stdlib in Python 3.11+; if you're on
   3.10 use `tomli` and adjust the import.

2. **`${VAR}` expansion doesn't recurse into nested dicts.** The TOML
   `[llm.backend] api_key = "${OPENAI_API_KEY}"` parses as
   `{"llm": {"backend": {"api_key": "${OPENAI_API_KEY}"}}}`. Your
   `_expand_env` helper needs to recurse. Tiri's
   `_substitute_env_vars` (in `tiri/config.py`) is the reference.

3. **The env-fallback path skips fields that exist in TOML.** When
   TOML and env both have a value, TOML wins for *shape*, env wins
   for *secrets*. The cleanest rule: env vars only fill values that
   TOML left as `""` or `None`. Don't try to merge dicts.

4. **Tests pass once, then fail because a real `OPENAI_API_KEY` leaks
   in from your shell.** Always use `monkeypatch.setenv` /
   `monkeypatch.delenv` in tests. Never rely on the absence of an
   env var in CI.

5. **You import `openai` at the top of `container.py`.** Tempting
   because Python is forgiving. Don't. Lazy import inside `_build_*`.

## Stretch

Add a second LLM backend type (whichever one you didn't pick in
Module 2 — Anthropic if you started with OpenAI, or vice versa).
Implement it as a separate `agent/providers/*_llm.py` file. Add the
branch in `_build_llm`. Run the "swap backends" test against both
real backends.

You have now built, in roughly 200 lines, the property that took
Tiri 6 fixmes to nail down across proxied APIs: same engine code,
different LLM vendors, no engine-side awareness of which is which.

## Reflection

1. Your config has ~5 fields right now. Tiri's has ~25. Look at
   `tiri/config.py` and ask: which of Tiri's fields would you *not*
   add to your agent, and why? (Hint: things like `intent_threshold`,
   `sql_max_retries`, `metadata_cache_ttl` are engine-tuning knobs that
   only matter once you have the engine they tune. Adding them now is
   premature.)

2. The "container always returns `RouterLLMProvider`" rule (which you
   haven't implemented yet, but Module 10 will) is paid for now in
   ~30 lines of indirection that buys zero behavior today. Is that
   worth it? In what kind of project would you say no?

3. Your `Config` is currently mutable (it's a plain dataclass). Tiri
   treats it as immutable by convention but doesn't enforce it. What
   are the arguments for and against making it `frozen=True`? When
   would the cost (you can't mutate it after construction) outweigh
   the benefit (no one in the codebase can sneak in a runtime change)?
