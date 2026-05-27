# Module 10 — Extensions: composability

> **The rule:** Extension beats rewrite. New capabilities are
> additive. Existing pipelines keep working.

By the end of this module your agent will route different tasks to
different LLM backends (cheap model for classification, smart model
for SQL), and will expose itself as a tool that *other* AI systems
can call via the Model Context Protocol. You won't build every
extension in Tiri's roadmap — that's 11 of them, and each is a
mini-project. The two you'll build here are the most teachable, and
they illustrate the larger pattern: each extension is *additive*,
not a rewrite. Existing code keeps working.

This is the last "build code" module. Module 11 is your capstone.

## Read

1. **`tiri/docs/extensions.md`** — the whole thing. There are 11
   extensions (EXT-1 through EXT-11). Read each one's summary
   paragraph; skim the implementation details. Focus on:
   - EXT-3 (multi-model routing) — we'll build a version of this.
   - EXT-4 (Tiri as an MCP server) and EXT-5 (Tiri as an MCP
     client) — we'll build a version of EXT-4.
   - EXT-1 (planning agent) and EXT-11 (hypothesis mode) — read
     the doc; we won't build them, but they're worth knowing about.

2. **`tiri/tiri/container.py`**, the `RouterLLMProvider` class — read
   in full. Notice: it implements `LLMProvider`, so consumers don't
   know it's a router. It just dispatches `complete()` calls to
   different underlying providers based on the `task` parameter.

3. **`tiri/docs/roadmap.md`** — skim. These are customer-validated
   capabilities tabled for future design. They're what Tiri's
   architecture is meant to support without rewrites.

## Concepts

### Why this module is short on new code

You have been writing code that supports extensions since Module 2.
Specifically:

- The `task` parameter on `LLMProvider.complete()` — added in
  Module 2, ignored by single-backend implementations.
- The `model` parameter on the same method — also added in Module 2.
- The container pattern — Module 3 always returns one consistent
  LLM type so swapping in a multi-backend implementation is a wiring
  change.
- The provider abstractions in general — every place we crossed an
  I/O boundary, we did it through an interface, which is what makes
  extensions like "swap the catalog for an MCP server" feasible.

Extensions in this course are *enabled by what you already built*.
You're not rewriting anything; you're wiring new providers into the
existing container.

### Multi-model routing, why

Different tasks have different cost/quality tradeoffs:

| Task | What it does | Right model |
|---|---|---|
| Intent classification | Pick from 3 categories | Small/cheap. The decision is small. |
| SQL generation | Write multi-table SQL | Big/smart. Real reasoning. |
| Synthesis | Plain-English summary | Medium. Coherent prose, not deep reasoning. |
| Embedding | Vectorize text | A dedicated embedding model. |

Running everything on `gpt-4o` is fine but wasteful. Running
everything on `gpt-4o-mini` is fast but the SQL gets worse. The
right answer is per-task routing: cheap for intent, big for SQL,
medium for synthesis.

The mechanism is the `task` parameter on `LLMProvider.complete()`.
Agents pass `task="intent"`, `task="sql"`, etc. A
`RouterLLMProvider` looks up the task in a config table and
forwards the call to the appropriate underlying provider. Single-
backend providers ignore the task entirely.

Your engine code does not change. Your agents do not change. Only
the container wiring changes.

### MCP, why

MCP (Model Context Protocol — defined in `Primer.md`) is the
standard for exposing tools and data to AI systems. There are two
sides:

- **Tiri as an MCP server.** Your agent is reachable from *any*
  MCP-aware client. Claude Desktop can ask your agent questions.
  Cursor can call your agent from inside an editor. Other agents
  (in different organizations, on different stacks) can compose
  your agent into a larger workflow.

- **Tiri as an MCP client.** Your agent can pull data from
  *external* MCP servers — a documentation server, a ticketing
  system, an internal search API. The agent gets richer context
  for questions where the answer isn't all in SQL.

Both are "extensions" in the precise sense: they're new surfaces on
the same engine. Your `RoomEngine.chat()` doesn't know whether the
question came from a curl command or an MCP `tools/call`. Your
agents don't know whether the context came from your catalog or
from an MCP server.

> **Concept: MCP server, recap.** A process that speaks the MCP
> protocol, exposing one or more *tools* (callable functions) and
> *resources* (readable data). MCP runs over stdio or HTTP. The
> protocol is JSON-RPC over either transport. Clients connect,
> discover tools, and call them by name with structured args.

### What you're not building

Tiri has 11 extensions. We're building variants of two
(`RouterLLMProvider` and an MCP server). Here's what we're skipping
and why:

- **EXT-1 (Planning Agent)** — multi-step questions that require
  multiple SQL queries. Conceptually important but a big build;
  read the doc, sketch it in your capstone if it's relevant.
- **EXT-2 (Wildcard table expansion)** — for rooms with thousands
  of tables. Cool, but not the bottleneck for most learners.
- **EXT-6 (Per-user UC enforcement)** — production-critical for
  multi-tenant Databricks deployments. Skip in local mode.
- **EXT-7 (Synthesis agent improvements)** — already covered in
  Module 9.
- **EXT-11 (Hypothesis mode)** — the controlled exception to the
  witness rule. Read `docs/extensions.md` EXT-11 and
  `docs/vision.md`'s hypothesis-mode section. If your domain wants
  this, it's a capstone-worthy project.

The pattern across all of them: each is additive. Each ships behind
a flag or via opt-in config. Existing rooms keep working.

## Lab

Four checkpoints. The first two add multi-model routing; the next
two expose your agent as an MCP server.

### Checkpoint 1 — Build `RouterLLMProvider`

Add `agent/providers/router_llm.py`:

```python
"""RouterLLMProvider — dispatches each task to a configured backend."""
from __future__ import annotations
from typing import AsyncIterator

from agent.data_models import LLMMessage, LLMResponse
from agent.providers.base import LLMProvider


class RouterLLMProvider(LLMProvider):
    def __init__(
        self,
        backends: dict[str, LLMProvider],
        routing: dict[str, str],
        default_backend: str,
    ) -> None:
        """
        backends: name → LLMProvider instance, e.g. {"cheap": OpenAILLMProvider(...), "smart": OpenAILLMProvider(...)}
        routing: task name → backend name, e.g. {"intent": "cheap", "sql": "smart"}
        default_backend: which backend to use when task isn't routed.
        """
        self._backends = backends
        self._routing = routing
        self._default = default_backend
        if default_backend not in backends:
            raise ValueError(
                f"default_backend {default_backend!r} not in backends"
            )

    def _resolve(self, task: str) -> LLMProvider:
        name = self._routing.get(task, self._default)
        if name not in self._backends:
            raise ValueError(
                f"task {task!r} routed to unknown backend {name!r}"
            )
        return self._backends[name]

    async def complete(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        max_tokens: int = 2048,
        task: str = "default",
        model: str | None = None,
    ) -> LLMResponse:
        backend = self._resolve(task)
        return await backend.complete(
            messages, temperature=temperature, max_tokens=max_tokens,
            task=task, model=model,
        )

    async def stream(
        self,
        messages: list[LLMMessage],
        temperature: float = 0.0,
        task: str = "default",
        model: str | None = None,
    ) -> AsyncIterator[str]:
        backend = self._resolve(task)
        async for chunk in backend.stream(
            messages, temperature=temperature, task=task, model=model,
        ):
            yield chunk

    async def embed(self, texts: list[str]) -> list[list[float]]:
        # Embedding is always routed to the "embed" task.
        backend = self._resolve("embed")
        return await backend.embed(texts)
```

This is ~50 lines. The pattern is straightforward: look up the
task, dispatch. Three things to notice:

1. **The router implements `LLMProvider` itself.** Consumers see an
   `LLMProvider`. They have no idea it's a router internally.

2. **Embedding is its own task.** Embedding models are different
   from completion models. Routing `task="embed"` to a dedicated
   embed backend is the standard pattern.

3. **Errors here are configuration errors.** A task routed to a
   nonexistent backend is a wiring bug — fail loud at startup, not
   silently at request time.

### Checkpoint 2 — Wire the router into the container

Update `Config` and `agent/container.py` to support multiple
backends.

`agent/config.py`:

```python
@dataclass
class LLMBackendConfig:
    name: str         # unique within Config.llm_backends
    type: str
    model: str
    api_key: str


@dataclass
class Config:
    llm_backends: list[LLMBackendConfig]
    llm_routing: dict[str, str] = field(default_factory=dict)
    default_backend: str = ""
    # ... existing fields ...
```

A real config file might look like:

```toml
[[llm.backends]]
name = "cheap"
type = "openai"
model = "gpt-4o-mini"
api_key = "${OPENAI_API_KEY}"

[[llm.backends]]
name = "smart"
type = "openai"
model = "gpt-4o"
api_key = "${OPENAI_API_KEY}"

[llm.routing]
intent = "cheap"
sql = "smart"
synthesis = "cheap"
embed = "cheap"
default = "cheap"
```

(Anthropic's equivalent if you have a Claude key: a second backend
with `type = "anthropic"`, `model = "claude-haiku-4-5-20251001"`.)

Then `agent/container.py`:

```python
def _build_llm(cfg: Config) -> LLMProvider:
    backends = {}
    for bc in cfg.llm_backends:
        backends[bc.name] = _instantiate_llm(bc)

    if len(backends) == 1 and not cfg.llm_routing:
        # Single backend, no routing — return it directly wrapped in
        # a trivial router. (Could return the backend itself, but
        # always returning the router type keeps consumers honest.)
        only = next(iter(backends))
        return RouterLLMProvider(
            backends=backends, routing={}, default_backend=only,
        )

    return RouterLLMProvider(
        backends=backends,
        routing=cfg.llm_routing,
        default_backend=cfg.default_backend or next(iter(backends)),
    )


def _instantiate_llm(bc: LLMBackendConfig) -> LLMProvider:
    if bc.type == "openai":
        from agent.providers.openai_llm import OpenAILLMProvider
        return OpenAILLMProvider(api_key=bc.api_key, model=bc.model)
    if bc.type == "anthropic":
        from agent.providers.anthropic_llm import AnthropicLLMProvider
        return AnthropicLLMProvider(api_key=bc.api_key, model=bc.model)
    raise ConfigurationError(f"unknown llm type: {bc.type!r}")
```

Restart the server. The engine and agents do not change. Every
LLM call now routes through your router. Verify with a benchmark
run:

```bash
python scripts/bench.py sales-room rooms/sales-room.benchmarks.json
```

If your routing has `"intent": "cheap"` and `"sql": "smart"`, your
LLM bill should drop noticeably. Quality on intent should stay the
same; quality on SQL should be the same or better.

Tests in `tests/test_router_llm.py`:

```python
import pytest
from agent.data_models import LLMMessage, LLMResponse
from agent.providers.base import LLMProvider
from agent.providers.router_llm import RouterLLMProvider


class _Tagged(LLMProvider):
    """Returns its own name as the response. Lets us verify routing."""
    def __init__(self, tag: str) -> None:
        self._tag = tag
    async def complete(self, messages, **kwargs):
        return LLMResponse(content=self._tag)
    async def stream(self, *a, **kw): raise NotImplementedError
    async def embed(self, texts):
        return [[float(len(t))] for t in texts]


@pytest.mark.asyncio
async def test_router_dispatches_by_task():
    router = RouterLLMProvider(
        backends={"a": _Tagged("a"), "b": _Tagged("b")},
        routing={"intent": "a", "sql": "b"},
        default_backend="a",
    )
    r1 = await router.complete([], task="intent")
    r2 = await router.complete([], task="sql")
    r3 = await router.complete([], task="unknown")     # → default
    assert r1.content == "a"
    assert r2.content == "b"
    assert r3.content == "a"


@pytest.mark.asyncio
async def test_router_raises_on_unknown_routed_backend():
    with pytest.raises(ValueError):
        RouterLLMProvider(
            backends={"a": _Tagged("a")},
            routing={"sql": "missing"},
            default_backend="a",
        ).complete([], task="sql")
```

Run:

```bash
pytest tests/test_router_llm.py -v
```

Both tests pass. You've shipped multi-model routing.

### Checkpoint 3 — Expose your agent as an MCP server

> **Concept: Why MCP for an agent.** Once your agent is an MCP
> server, *any* MCP-aware client can call it. Claude Desktop has a
> tool list; your agent appears in it. Cursor can pull data from
> your agent into a code session. Another team's agent — running
> entirely separately — can include yours as a step in its
> workflow. You haven't built a UI for any of these; the protocol
> handles the integration.

Install the MCP SDK:

```bash
pip install mcp
```

Create `agent/api/mcp/__init__.py` and `agent/api/mcp/server.py`:

```python
"""Expose the agent as an MCP server. One tool: 'ask'."""
from __future__ import annotations
import json
import uuid

from mcp.server import Server
from mcp.types import Tool, TextContent

from agent.engine.room_engine import RoomEngine


def build_mcp_server(engine: RoomEngine, default_room: str) -> Server:
    """Wire the engine as an MCP server with a single 'ask' tool."""
    server = Server("my-agent")

    @server.list_tools()
    async def list_tools() -> list[Tool]:
        return [
            Tool(
                name="ask",
                description=(
                    "Ask a data question. The agent will retrieve from "
                    "the configured room and return an answer with "
                    "evidence and caveats."
                ),
                inputSchema={
                    "type": "object",
                    "properties": {
                        "question": {
                            "type": "string",
                            "description": "The natural-language question.",
                        },
                        "room_id": {
                            "type": "string",
                            "description": (
                                f"Optional room override; defaults to "
                                f"{default_room!r}."
                            ),
                        },
                    },
                    "required": ["question"],
                },
            )
        ]

    @server.call_tool()
    async def call_tool(name: str, arguments: dict) -> list[TextContent]:
        if name != "ask":
            raise ValueError(f"unknown tool: {name!r}")
        question = arguments["question"]
        room_id = arguments.get("room_id", default_room)
        turn = await engine.chat(
            room_id=room_id,
            conversation_id=f"mcp-{uuid.uuid4()}",
            question_text=question,
        )

        # Compact JSON payload — the MCP client surfaces this to its
        # own model.
        payload = {
            "question": question,
            "room_id": room_id,
        }
        if turn.answer:
            payload["answer"] = turn.answer.text
            payload["sql"] = (turn.answer.metadata or {}).get("sql", "")
            payload["caveats"] = turn.answer.caveats
        elif turn.unable_to_answer:
            payload["unable_to_answer"] = turn.unable_to_answer
        elif turn.clarification_question:
            payload["clarification"] = turn.clarification_question

        return [TextContent(type="text", text=json.dumps(payload, indent=2))]

    return server
```

Notice the *shape* of the MCP server: one `list_tools()` decorator
that declares what's available, one `call_tool()` decorator that
implements the calls. The tool's `inputSchema` is a JSON schema —
MCP-aware clients use it to construct calls and validate input.

The output is `TextContent` carrying compact JSON. MCP supports
richer content types (images, embedded resources), but for an
agent that returns structured answers, JSON text is enough.

### Checkpoint 4 — Run the MCP server and call it

The MCP SDK ships a stdio runner. Add `scripts/mcp_server.py`:

```python
"""Run the agent as an MCP server over stdio."""
import asyncio
from mcp.server.stdio import stdio_server

from agent.config import Config
from agent.container import build_container
from agent.api.mcp.server import build_mcp_server


async def main():
    cfg = Config.load()
    container = build_container(cfg)
    engine = container["engine"]
    server = build_mcp_server(engine, default_room="sales-room")

    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream, write_stream,
            initialization_options=server.create_initialization_options(),
        )


if __name__ == "__main__":
    asyncio.run(main())
```

Test it with the MCP SDK's CLI:

```bash
pip install "mcp[cli]"
mcp dev scripts/mcp_server.py
```

`mcp dev` is an interactive dev UI for MCP servers. You'll see your
server connect, your `ask` tool listed, and an interactive prompt
where you can call it. Try:

- `ask({"question": "What was total revenue by region?"})`
- `ask({"question": "Why is revenue declining?"})`   ← refusal path
- `ask({"question": "tell me about sales"})`         ← clarification path

The output is the JSON payload from `call_tool()`. Each invocation
runs your full pipeline.

**Optional: hook into Claude Desktop.** Add to your Claude Desktop
config (`~/Library/Application Support/Claude/claude_desktop_config.json`
on macOS):

```json
{
  "mcpServers": {
    "my-agent": {
      "command": "python",
      "args": ["/absolute/path/to/scripts/mcp_server.py"],
      "env": {
        "OPENAI_API_KEY": "sk-...",
        "PYTHONPATH": "/absolute/path/to/my-agent"
      }
    }
  }
}
```

Restart Claude Desktop. Open a conversation. Type `/`. Your `ask`
tool should appear in the tool picker. Pick it; pass a question.
Claude calls your agent, gets the result, and incorporates it into
the conversation.

Your data agent is now usable from Claude Desktop, Cursor, or any
other MCP client — without you writing a single client-side line.

### Definition of done

- `agent/providers/router_llm.py` defines `RouterLLMProvider` and
  is wired through the container.
- A benchmark run uses at least two distinct underlying models
  (intent on cheap, SQL on smart).
- `agent/api/mcp/server.py` exposes an `ask` MCP tool.
- `scripts/mcp_server.py` runs over stdio.
- `mcp dev scripts/mcp_server.py` lets you call the `ask` tool
  interactively and returns answers from your room.
- Commit:
  ```bash
  git add . && git commit -m "module-10: multi-model routing + MCP server"
  ```

### Common pitfalls

1. **The `RouterLLMProvider` looks like over-engineering for a
   single-backend setup.** It is — in isolation. In context, it's
   the difference between "Module 10 is a wiring change" and
   "Module 10 is a refactor across every agent." Pay the small
   cost early.

2. **You wire the router but the routing config is empty.** Every
   request goes to the default backend. The router does nothing
   useful. Symptom: cost stays the same after Module 10. Verify
   your `llm.routing` table is populated and your agents are
   passing `task="..."` correctly.

3. **MCP `inputSchema` doesn't match what your code reads.** The
   schema says `room_id` is optional; your code reads
   `arguments["room_id"]` without `.get()` and crashes when the
   client omits it. Test with both forms.

4. **`mcp dev` works but Claude Desktop doesn't see the server.**
   Most common cause: the `command` in the config is `python` but
   Claude Desktop's PATH doesn't include your venv's Python. Use
   the absolute path: `command: "/full/path/to/.venv/bin/python"`.

5. **You expose the MCP `ask` tool with no description.** The
   model on the client side decides whether to call your tool
   based on the description and name. A vague description ("Ask
   the agent") means the model will rarely pick your tool when
   others are available. Spell out what your agent excels at —
   the refusal manifesto from Module 0 is good source material.

## Stretch

1. **Build an MCP *client*.** Pull data from an external MCP server
   into your agent's context. The simplest version: at request
   time, call a tool on an external MCP server (a documentation
   server, a calendar API, anything you have access to) and append
   its result to the `ContextPackage`. Surface it to the SQL agent
   as additional context.

2. **Build a planning agent (EXT-1).** Read `docs/extensions.md`
   EXT-1. The shape: before the SQL agent, a `PlanningAgent` looks
   at the question and produces a *plan* — a list of sub-queries
   needed to answer it. Each sub-query goes through the SQL agent
   independently; the synthesis agent combines the results.
   Significant build — but it unlocks the "year-over-year
   comparison" class of question that one query can't answer.

3. **Per-task model overrides at request time.** The `model`
   parameter we added in Module 2 lets a single request specify a
   model. Wire the API to accept a `?model_override=` query
   parameter and pass it through. Use it for side-by-side
   comparisons in the benchmark runner — same question, two
   models, which one is right?

4. **Streaming MCP.** MCP supports streaming responses. Refactor
   `call_tool()` to yield partial results as your `stream_chat()`
   does. The client can show progress while the agent works.

## Reflection

1. The `RouterLLMProvider` is a generic dispatch table — task →
   backend. What other kinds of routing might a real system want?
   (Hint: by user permission, by data sensitivity, by time of day
   for cost control.) Sketch the API; don't implement.

2. Exposing your agent as an MCP server makes it composable with
   other AI systems. It also means a client might invoke your
   agent in contexts your refusal manifesto didn't anticipate. How
   do you reconcile "the agent refuses to answer X" with "another
   agent calls mine and re-frames the question to dodge the
   refusal"? Is this a technical problem or a governance problem?

3. Tiri's 11 extensions are roadmapped, not all built. What does
   that say about the architecture? (Hint: each extension is
   additive to the same core. If extensions required a rewrite
   each time, the architecture would have collapsed by EXT-3.)

4. Pick *one* extension you'd build for your domain. Write a
   one-paragraph proposal: what does it add, what existing code
   does it touch, what doesn't change? This is the seed of your
   capstone project (Module 11).
