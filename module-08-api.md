# Module 8 — Surface: REST + SSE streaming

> **The rule:** Trust comes from transparency in the protocol, not
> chrome. The wire format is part of the product.

In this module you'll put a thin HTTP layer in front of your
`RoomEngine`. By the end you'll have a working FastAPI server that
serves both a request/response endpoint and a streaming endpoint
that emits intermediate events as the pipeline runs. A user
watching the stream sees the SQL before they see the answer — which
turns out to be one of the highest-leverage features for trust.

This module is short on concepts and long on plumbing. Most of it
is FastAPI patterns you may have seen before; the interesting parts
are the *streaming protocol* and *what events the engine emits*.

## Read

1. **`tiri/docs/api.md`** — the whole thing. Pay attention to:
   - The endpoint list (POST a question, GET a stream, list turns,
     manage rooms).
   - The SSE event types (`status`, `sql`, `result`, `viz`,
     `synthesis`, `done`, `error`). Each event has a structured
     shape.

2. **`tiri/tiri/api/main.py`** — read the `create_app()` function.
   Notice how the container is attached to `app.state` once at
   startup, and how routes pull providers off it via dependency
   injection.

3. **`tiri/tiri/api/routes/conversations.py`** — read the `send_message`
   and `stream_messages` endpoints. ~50 lines for the streaming
   endpoint; the engine does the work.

## Concepts

### REST: the boring half

> **Concept: REST.** A convention for HTTP APIs where URLs name
> *resources* (conversations, rooms, messages), HTTP verbs name
> *actions* (POST creates, GET reads, PUT updates, DELETE removes),
> and the response body carries the resource's state. There's no
> "REST framework" — it's a set of conventions and a basis to
> argue with colleagues over for two hours.

The agent's REST shape is small:

| Method | Path | What it does |
|---|---|---|
| `POST` | `/rooms/{room_id}/conversations` | Start a new conversation |
| `POST` | `/rooms/{room_id}/conversations/{conv_id}/messages` | Ask a question, wait for the full answer |
| `GET` | `/rooms/{room_id}/conversations/{conv_id}/messages/stream` | Ask a question, get a live stream of events |
| `GET` | `/rooms/{room_id}/conversations/{conv_id}/messages` | List all turns in a conversation |

The non-streaming POST is what most clients call. The streaming GET
is what a UI uses to give the user live feedback.

### SSE: the interesting half

> **Concept: SSE (Server-Sent Events).** A simple protocol where
> the server holds an HTTP response open and writes named events to
> it as they happen. The client subscribes by making a regular
> GET request; it doesn't need WebSockets. The wire format is just
> `data: <text>\n\n` per event. Browsers have a built-in
> `EventSource` API for it.

Why SSE for an agent? Because the agent's value is in *showing the
work*, and the user wants to see the work as it happens:

```
0.0s  status: Loading room configuration...
0.2s  status: Building context...
0.8s  status: Classifying question...
1.4s  intent: answerable
1.6s  status: Generating SQL...
3.2s  sql: SELECT region, SUM(amount) FROM orders GROUP BY region
3.4s  status: Validating...
3.5s  status: Executing...
4.1s  result: 4 rows
5.6s  synthesis: "North led with $1.2M..."
5.7s  done
```

A non-streaming endpoint would return the final blob at 5.7s. The
streaming version reveals the SQL at 3.2s — *before* the answer.
That's not just UX polish. It's a trust mechanism: the user can see
the query that produced their answer at the moment it's generated,
and challenge it if it looks wrong. The 2 seconds between "SQL"
and "synthesis" is often enough for a careful user to spot a bad
query before they accept the answer derived from it.

This is the same reasoning behind "show the SQL" in the final
response. Streaming is "show the SQL *early*."

### Why FastAPI

> **Concept: FastAPI.** A Python HTTP framework built on top of
> Starlette and Pydantic. Async-first, type-driven (your function
> annotations become the API schema), with a developer experience
> that compounds nicely. Not the only choice — Flask, Django,
> Sanic, and aiohttp all work — but a strong default for new
> Python services.

Three FastAPI features the agent uses:

1. **Async route handlers** — every endpoint is `async def` so
   waiting on the LLM doesn't block other requests.
2. **Path and query parameters** — `room_id: str` and
   `question: str` in the signature become URL pieces FastAPI
   parses automatically.
3. **Dependency injection** — `Depends(_engine)` pulls the
   `RoomEngine` off `app.state` for each request. The same pattern
   that wired the container in Module 3.

You don't need to know FastAPI deeply to use it for this module.
The patterns repeat.

### Authentication: a brief note, then deferred

A real agent serving multiple users needs per-user authentication
and (often) per-user data access via UC permissions. Tiri does
both: the `Authorization` header is parsed into a user token, and
that token threads through every provider call.

This course defers authentication. The lab uses unauthenticated
endpoints with `auth_disabled=True`. That's fine for local
development; it's *not* okay for anything that touches real user
data. When you wire your agent into a production environment, the
auth story is mandatory homework — start from `tiri/api/auth.py`
and the EXT-6 per-user credentials work in `docs/extensions.md`.

## Lab

Six checkpoints. By the end you'll have a server, a client, and a
streaming pipeline that emits live events.

### Checkpoint 1 — Install FastAPI and scaffold the app

```bash
pip install "fastapi>=0.115" "uvicorn[standard]>=0.30"
```

```bash
mkdir -p agent/api/routes
touch agent/api/__init__.py agent/api/main.py
touch agent/api/routes/__init__.py agent/api/routes/conversations.py
```

`agent/api/main.py`:

```python
"""FastAPI application factory."""
from __future__ import annotations
import logging
from fastapi import FastAPI
from fastapi.responses import JSONResponse

from agent.config import Config
from agent.container import build_container


log = logging.getLogger("agent.api")


def create_app(config: Config | None = None) -> FastAPI:
    cfg = config or Config.load()
    container = build_container(cfg)

    app = FastAPI(
        title="My Agent",
        version="0.1.0",
        description="A trustworthy data agent.",
    )
    # Attach container so route handlers can pull providers off it.
    app.state.container = container

    # Routes.
    from agent.api.routes.conversations import router as conv_router
    app.include_router(conv_router, prefix="/rooms")

    # Global exception handler: every unhandled error becomes a
    # JSON response, never an HTML stack trace.
    @app.exception_handler(Exception)
    async def _handle_unexpected(request, exc):
        log.exception("unhandled error: %s", exc)
        return JSONResponse(
            status_code=500,
            content={"error": "internal", "message": str(exc)},
        )

    return app


app = create_app()   # module-level for `uvicorn agent.api.main:app`
```

Notice three things:

1. `create_app()` is a factory. Tests can call it with a custom
   `Config` (mocked container) without a real one having to exist.
2. The container goes on `app.state`, not as a module-level global.
3. The global exception handler turns crashes into JSON. A UI
   client should never see an HTML stack trace.

### Checkpoint 2 — The non-streaming endpoint

`agent/api/routes/conversations.py`:

```python
"""Conversation endpoints — non-streaming + streaming."""
from __future__ import annotations
import json
import uuid
from dataclasses import asdict
from typing import Any, AsyncIterator

from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import StreamingResponse

from agent.engine.room_engine import RoomEngine


router = APIRouter(tags=["conversations"])


def _engine(request: Request) -> RoomEngine:
    """Pull the engine off app.state. Dependency-injection style."""
    return request.app.state.container["engine"]


@router.post("/{room_id}/conversations", status_code=201)
async def start_conversation(room_id: str) -> dict[str, Any]:
    """Allocate a conversation id. Stateless — no DB write yet."""
    return {"conversation_id": str(uuid.uuid4()), "room_id": room_id}


@router.post("/{room_id}/conversations/{conv_id}/messages")
async def send_message(
    request: Request,
    room_id: str,
    conv_id: str,
    body: dict[str, Any],
) -> dict[str, Any]:
    """Ask a question; wait for the full answer."""
    question = body.get("question")
    if not isinstance(question, str) or not question.strip():
        raise HTTPException(
            status_code=422,
            detail={"error": "validation_error",
                    "message": "`question` must be a non-empty string"},
        )
    engine = _engine(request)
    turn = await engine.chat(
        room_id=room_id,
        conversation_id=conv_id,
        question_text=question,
    )
    return asdict(turn)
```

Run:

```bash
uvicorn agent.api.main:app --reload
```

Expected output:

```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [...]
INFO:     Started server process [...]
INFO:     Application startup complete.
```

Test it from a second terminal:

```bash
# Start a conversation
curl -X POST http://127.0.0.1:8000/rooms/sales-room/conversations
# → {"conversation_id":"<uuid>","room_id":"sales-room"}

# Ask a question (substitute the conversation_id)
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What was total revenue by region?"}' \
  http://127.0.0.1:8000/rooms/sales-room/conversations/<conv_id>/messages
```

Expected output (one big JSON blob, the `Turn`):

```json
{
  "turn_id": "...",
  "question": {"question_id": "...", "text": "What was total revenue by region?", ...},
  "answer": {
    "text": "4 rows across 2 columns: region, sum(amount).",
    "evidence": [{"source": "main.orders", "excerpt": "4 rows returned", "confidence": "high"}],
    "caveats": [...]
  },
  "unable_to_answer": null,
  "clarification_question": null
}
```

That's your `RoomEngine.chat()` from Module 7, now reachable over
HTTP.

### Checkpoint 3 — The streaming endpoint

This requires `RoomEngine.stream_chat()` — a variant of `chat()`
that yields events as it runs. Add to `agent/engine/room_engine.py`:

```python
from typing import AsyncIterator


class RoomEngine:
    ...

    async def stream_chat(
        self,
        room_id: str,
        conversation_id: str,
        question_text: str,
    ) -> AsyncIterator[dict]:
        """Run the pipeline, yielding events as it progresses.

        Mirrors chat() but emits intermediate events. Errors are
        emitted as {"type": "error", "message": ...} rather than
        raised — the SSE stream stays open until "done" or "error".
        """
        try:
            yield {"type": "status", "text": "Loading room configuration..."}
            config = await self._store.get_room_config(room_id)

            yield {"type": "status", "text": "Building context..."}
            context = await self._build_context(question_text, config)

            yield {"type": "status", "text": "Classifying question..."}
            intent = await IntentAgent(self._llm).run(question_text, context)
            yield {"type": "intent", "kind": intent.kind, "reasoning": intent.reasoning}

            if intent.kind == "clarification_needed":
                yield {
                    "type": "clarification",
                    "text": intent.clarification or "Could you provide more detail?",
                }
                yield {"type": "done"}
                return

            if intent.kind == "refused":
                yield {
                    "type": "refused",
                    "text": intent.refusal_reason
                            or "I cannot answer that with the available data.",
                }
                yield {"type": "done"}
                return

            yield {"type": "status", "text": "Generating SQL..."}
            sql_result = await SQLAgent(self._llm, self._query).run(
                question_text, context, intent,
            )
            if not sql_result.is_valid:
                yield {"type": "error", "message": f"SQL generation failed: {sql_result.error}"}
                yield {"type": "done"}
                return
            yield {"type": "sql", "sql": sql_result.sql, "attempts": sql_result.attempts}

            yield {"type": "status", "text": "Executing..."}
            query_result = await self._query.execute(sql_result.sql)
            yield {
                "type": "result",
                "columns": query_result.columns,
                "row_count": query_result.row_count,
                "rows": query_result.rows[:50],   # cap preview at 50
            }

            yield {"type": "status", "text": "Synthesizing answer..."}
            summary = _synthesize_one_line(query_result)
            yield {"type": "synthesis", "text": summary}

            yield {"type": "done"}

        except Exception as e:
            _log.exception("stream_chat failed")
            yield {"type": "error", "message": str(e)}
            yield {"type": "done"}
```

Two things to notice:

1. Every step yields a `status` event before doing work. Cheap to
   produce, hugely useful for the watching user.
2. Errors are yielded as `error` events, then `done`. The stream
   never raises out of the generator. A swallowed error here would
   be invisible to the user; an unhandled raise would crash the
   connection mid-stream.

Now the SSE endpoint. Add to `conversations.py`:

```python
@router.get("/{room_id}/conversations/{conv_id}/messages/stream")
async def stream_messages(
    request: Request,
    room_id: str,
    conv_id: str,
    question: str,
) -> StreamingResponse:
    """SSE stream. Question comes via query parameter (no GET body)."""
    if not question.strip():
        raise HTTPException(
            status_code=422,
            detail={"error": "validation_error",
                    "message": "`question` query param must be non-empty"},
        )
    engine = _engine(request)

    async def event_source() -> AsyncIterator[str]:
        async for event in engine.stream_chat(
            room_id=room_id,
            conversation_id=conv_id,
            question_text=question,
        ):
            yield f"data: {json.dumps(event)}\n\n"

    return StreamingResponse(event_source(), media_type="text/event-stream")
```

The `data: <json>\n\n` format is the literal SSE wire protocol.
Each event is one `data:` line followed by a blank line.

### Checkpoint 4 — Test the stream from curl

Start the server (already running with `--reload`, or restart it):

```bash
uvicorn agent.api.main:app --reload
```

In another terminal:

```bash
curl -N "http://127.0.0.1:8000/rooms/sales-room/conversations/c1/messages/stream?question=What%20was%20total%20revenue%20by%20region%3F"
```

`-N` disables curl's output buffering so events appear in real time.

Expected output (one event per line, with timing):

```
data: {"type": "status", "text": "Loading room configuration..."}

data: {"type": "status", "text": "Building context..."}

data: {"type": "status", "text": "Classifying question..."}

data: {"type": "intent", "kind": "answerable", "reasoning": "..."}

data: {"type": "status", "text": "Generating SQL..."}

data: {"type": "sql", "sql": "SELECT region, SUM(amount) FROM orders GROUP BY region", "attempts": 1}

data: {"type": "status", "text": "Executing..."}

data: {"type": "result", "columns": ["region", "sum(amount)"], "row_count": 4, "rows": [["NORTH", 62.49], ...]}

data: {"type": "status", "text": "Synthesizing answer..."}

data: {"type": "synthesis", "text": "4 rows across 2 columns: region, sum(amount)."}

data: {"type": "done"}
```

Watching the events appear is the first time you'll *feel* the
pipeline. Each one corresponds to one step in your engine. If a step
is slow, you see exactly which one.

### Checkpoint 5 — Tests against the API

`tests/test_api.py`:

```python
"""API tests using FastAPI's TestClient."""
from __future__ import annotations
import json
import pytest
from fastapi.testclient import TestClient

from agent.api.main import create_app
from agent.config import Config, LLMBackendConfig


@pytest.fixture
def app(tmp_path, monkeypatch):
    """Build an app pointed at a temp directory for state."""
    monkeypatch.setenv("OPENAI_API_KEY", "sk-test")
    cfg = Config(
        llm_backend=LLMBackendConfig(
            type="openai", model="gpt-4o-mini", api_key="sk-test",
        ),
        query_data_dir=str(tmp_path / "data"),
        store_dir=str(tmp_path / "rooms"),
    )
    return create_app(config=cfg)


def test_start_conversation_returns_id(app):
    client = TestClient(app)
    resp = client.post("/rooms/room-1/conversations")
    assert resp.status_code == 201
    body = resp.json()
    assert "conversation_id" in body
    assert body["room_id"] == "room-1"


def test_send_message_rejects_empty_question(app):
    client = TestClient(app)
    resp = client.post(
        "/rooms/room-1/conversations/c1/messages",
        json={"question": ""},
    )
    assert resp.status_code == 422
    assert resp.json()["detail"]["error"] == "validation_error"


def test_send_message_rejects_missing_question(app):
    client = TestClient(app)
    resp = client.post(
        "/rooms/room-1/conversations/c1/messages",
        json={"not_question": "hi"},
    )
    assert resp.status_code == 422
```

These tests don't run the full pipeline — they validate the API
*surface*. Pipeline behavior is tested at the agent and engine
levels (Modules 6–7); benchmark behavior is tested in Module 9.
Don't double-test the pipeline through the API.

Run:

```bash
pytest tests/test_api.py -v
```

Expected output:

```
tests/test_api.py::test_start_conversation_returns_id PASSED
tests/test_api.py::test_send_message_rejects_empty_question PASSED
tests/test_api.py::test_send_message_rejects_missing_question PASSED
```

### Checkpoint 6 — Optional: minimal HTML viewer

If you want to see the streaming in a browser, drop this into
`viewer.html` and open it:

```html
<!DOCTYPE html>
<html>
<head><title>Agent stream</title></head>
<body>
  <input id="q" size="60" placeholder="Ask a question..."/>
  <button onclick="ask()">Ask</button>
  <pre id="out"></pre>
  <script>
    function ask() {
      const out = document.getElementById('out');
      out.textContent = '';
      const q = encodeURIComponent(document.getElementById('q').value);
      const url = `http://127.0.0.1:8000/rooms/sales-room/conversations/c1/messages/stream?question=${q}`;
      const es = new EventSource(url);
      es.onmessage = (e) => {
        const event = JSON.parse(e.data);
        out.textContent += `[${event.type}] ` + JSON.stringify(event) + '\n';
        if (event.type === 'done' || event.type === 'error') es.close();
      };
      es.onerror = () => es.close();
    }
  </script>
</body>
</html>
```

Open in a browser, type a question, click Ask. Watch the events
stream in. Note: you'll need CORS headers on the API for this to
work from a `file://` URL — install `python-multipart` and add the
CORSMiddleware to `create_app()` if you want this to work:

```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # for local dev; lock down in prod
    allow_methods=["*"],
    allow_headers=["*"],
)
```

This is optional. Most of the lab's value is the curl version —
the HTML is icing.

### Definition of done

- `uvicorn agent.api.main:app` runs without errors.
- `POST /rooms/sales-room/conversations` returns a 201 with a
  conversation id.
- `POST /rooms/sales-room/conversations/<id>/messages` returns a
  full Turn JSON.
- `GET /rooms/sales-room/conversations/<id>/messages/stream?question=...`
  emits events as `text/event-stream`.
- API tests pass.
- Commit:
  ```bash
  git add . && git commit -m "module-08: FastAPI surface with SSE streaming"
  ```

### Common pitfalls

1. **The stream looks buffered, not live.** Two causes: (a) the
   client buffers (use `curl -N`), (b) a reverse proxy buffers
   (nginx with `proxy_buffering on` will hold the whole stream
   until done). For development, just curl directly to uvicorn and
   you're fine.

2. **Errors mid-stream leak the stack trace.** The `try/except` in
   `stream_chat` catches everything and yields a `{"type":
   "error"}` event. Any unhandled raise in a generator function
   that's been wrapped in `StreamingResponse` will look like a
   silent disconnect to the client. Wrap the body.

3. **`StreamingResponse` doesn't include CORS headers.** If you're
   serving a UI from a different origin, the browser will silently
   refuse to read the stream. Add `CORSMiddleware`.

4. **You write `yield f"data: {event}\n\n"` with a Python repr
   instead of JSON.** The client gets `{'type': 'status'}` (single
   quotes), which is not valid JSON, and `JSON.parse` fails. Use
   `json.dumps(event)`.

5. **`uvicorn --reload` doesn't reload your prompt template file
   changes.** It watches `.py` files by default. Either restart
   manually after prompt edits or add `--reload-dir agent/engine`.

6. **Tests pass but the streaming endpoint doesn't actually stream
   when you curl.** Look at your `event_source()` — it must be an
   async generator (uses `yield`), not return a list. A list-
   returning function makes `StreamingResponse` buffer the whole
   thing.

## Stretch

1. **Add a `GET /rooms/{room_id}/conversations/{conv_id}/messages`
   endpoint** that lists all turns persisted for a conversation.
   Requires extending `FileStoreProvider` to persist turns and
   index them by conversation. Test that ask → list returns the
   turn you just asked for.

2. **Add a heartbeat to the stream.** Long-running SQL queries
   (>30s) can cause middleware (nginx, AWS load balancers) to drop
   the connection. Yield a `{"type": "heartbeat"}` event every 15
   seconds. The simplest implementation is a parallel
   `asyncio.create_task` for the pipeline and a heartbeat loop;
   merge their outputs.

3. **Authentication.** Add an `Authorization: Bearer <token>` header
   requirement. Wire the token through the engine (Module 7's
   `chat()` already accepts `user_token`; you'd add it). Don't
   build a full auth system — just verify the token is non-empty
   and pass it through. Real auth is operational work for whatever
   identity provider you target.

## Reflection

1. The streaming endpoint emits `status` events for every step —
   "Loading room config", "Building context", etc. These cost
   nothing to add and matter a lot for perceived performance. What
   non-obvious cost might they impose, and when would you remove
   them? (Hint: think about what a sophisticated user might learn
   from seeing every step that an unsophisticated one would not.)

2. The non-streaming POST returns one big JSON blob with the
   complete `Turn`. The streaming GET emits intermediate events.
   They're not interchangeable — neither is strictly better. When
   would a client prefer one over the other?

3. Tiri's stream emits events for `sql`, `result`, `viz`, and
   `synthesis` separately, in that order. Why this order and not
   `synthesis` first? (Hint: trust mechanism. The user can see the
   SQL while waiting for the synthesis to finish, and challenge a
   bad query before the answer commits to it.)

4. The API has zero authentication today. For your domain (the
   manifesto from Module 0), what is the smallest auth story that
   would let you run this in front of real users? Token check?
   Per-user data access? Audit log? Sketch the surface; don't
   build it.
