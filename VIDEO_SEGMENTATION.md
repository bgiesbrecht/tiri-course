# Suggested video segmentation

A plan for turning the written course into a video series. The
written modules stay as they are — the *labs*, with checkpoints and
skeleton code. The videos carry the *narrative*, the *concepts*,
and the *demo moments* that don't read well on a page.

## Production philosophy

Three rules that shape every decision below:

1. **Don't film labs live.** Live coding for a 7-checkpoint module
   is 90 minutes of tape, half of it spent waiting on `pip install`.
   The video covers the *idea*; the viewer does the lab afterward
   with the written module as the script. Videos are 10–25 minutes;
   labs are 1–3 hours.

2. **Demo the moments that earn it.** Each arc has one or two
   payoffs that *only video can deliver* — watching the streaming
   pipeline emit events, calling your agent from Claude Desktop,
   seeing a benchmark fail and walking through the tuning. Film
   those. Don't pad with screen recordings of file creation.

3. **Concepts before code.** Every video opens with the concept and
   the design rule. Code clips are illustrations, not the lecture.
   A viewer should be able to watch the video on a phone, on a
   train, *without* a keyboard, and still get most of the value.

## Total runtime estimate

13 videos. **~5.5 hours of finished video** in total. Plus an
optional 30-min GenAI primer for viewers without LLM background.

Self-paced lab time across all modules: another 30–40 hours. The
videos are the lecture half; the labs are the studio half.

---

## The video series

### 0. Welcome / Course frame (10 min)

The why. Not the how.

**Cover:**
- What you'll build by the end (show a clip of the final agent
  answering a question, refusing one, asking a clarifying question)
- The thesis: "trust comes from architecture, not personality"
- Who this is for / not for
- How to use the course (videos + written modules + labs +
  Primer.md)
- The big-picture diagram from the README, narrated

**Demo moment:** A 30-second cold-open clip of the finished agent
in action — answering, refusing, streaming the SQL. Set the
destination before naming the steps.

**Pairs with:** README.md.

---

### 1. (Optional) GenAI primer (25–30 min)

For viewers who don't yet know what an LLM is, what embeddings
are, what RAG is. **Skippable** for anyone with prior GenAI
experience; this is why it's optional.

**Cover:**
- LLMs from the outside: text in, text out, no memory between calls
- Prompts, tokens, context windows
- Temperature, determinism, hallucination
- Embeddings and vector search (with a visual — two sentences,
  their vectors, the cosine distance)
- RAG: the pattern in one minute
- Agent vs chatbot vs tool use
- What MCP is and why it's interesting

**Demo moment:** Open the OpenAI playground or equivalent; show
the same question at `temperature=0` and `temperature=1`. The same
question, three different answers, then three identical ones.

**Pairs with:** Primer.md.

---

### 2. Why this agent refuses things (15 min)

The frame. Set up the rest of the course.

**Cover:**
- "Tiri is a witness, not an analyst"
- Retrieval questions vs reasoning questions
- Who the audience is (congressional staffers, CEOs, managers,
  not chatbot users)
- Why fluent confidence is dangerous in those settings
- What the refusal manifesto is and why it goes first
- Walk through one worked example: the manifesto for a "renewal
  conversation" agent from Module 0

**Demo moment:** Read the refusal manifesto example out loud,
then show the finished agent refusing one of the manifest's
forbidden claim types.

**Pairs with:** Module 00 (lab: write your own manifesto).

---

### 3. Data models as contracts (15 min)

First arc of the foundation.

**Cover:**
- Dataclasses vs dicts: where invariants live
- The mutual-exclusion pattern (`__post_init__` raising on
  invalid state at construction)
- Why Python's `@dataclass`, not Pydantic, for *internal* models
- "Mistakes happen at the boundary, not the middle"

**Demo moment:** Open the Tiri repo. `grep` for `__post_init__`
across `data_models.py`. Read three of them aloud — show how each
one encodes a real constraint, not paranoia. Then deliberately
violate one in a Python REPL and let it raise.

**Pairs with:** Module 01.

---

### 4. The provider boundary (20 min)

The architectural decision that makes everything else work.

**Cover:**
- "Engine has zero I/O" — the rule, stated and motivated
- The seven providers (LLM, Catalog, Metadata, Query, Vector,
  Store, MCP) introduced with one sentence each
- Abstract base classes as *contracts in code*
- `validate()` before `execute()` — why every SQL call is parsed
  before it runs
- The cost: more code, more interfaces. The benefit: testable,
  swappable, mockable.

**Demo moment:** Run a Tiri unit test suite live. 467 tests in
3 seconds, no network. Then `grep -rn "import openai\|import
databricks" tiri/engine/` and show zero results. The architecture
*claims* zero I/O in the engine; the grep proves it.

**Pairs with:** Module 02.

---

### 5. Configuration and the container (15 min)

The shortest arc, but a key inflection point.

**Cover:**
- TOML for shape; env vars for secrets. Why both, why not one.
- The container pattern: one function takes `Config`, returns a
  dict of providers
- Lazy imports for optional dependencies
- The "always return `RouterLLMProvider`" rule — pay 30 lines now,
  save a refactor later

**Demo moment:** Show two `agent.toml` files that differ only in
the LLM backend. Run the agent against both. Same code path; same
behavior. The architecture's promise made literal.

**Pairs with:** Module 03.

---

### 6. Knowledge: how the agent finds context (25 min)

The first GenAI-rich arc. RAG is here.

**Cover:**
- The `ContextPackage` as a unit of work: "before any agent runs,
  here is everything it might need"
- The metadata stack: physical catalog → semantic YAML → room
  overrides. Merge rules and conflict tracking.
- Embeddings, visually. A sentence → a vector → the nearest
  neighbors. With a real demo.
- RAG as a pattern: embed the question, retrieve top-k examples,
  stuff them into the prompt.
- Why retrieval beats stuffing the whole catalog

**Demo moment:** Open a Chroma collection in a notebook. Index 6
example questions. Run a 7th question through `query()`. Show
which example came back as the nearest neighbor — *and why*.
Embedding similarity, made visible.

**Pairs with:** Module 04.

---

### 7. Prompts as files, not f-strings (12 min)

Short arc; big payoff.

**Cover:**
- The four things that break with inline f-strings
- The structure of a working prompt: role, constraints, sections,
  examples, question, format reminder
- System prompts vs user/assistant messages
- "Prompt engineering" — what it isn't (magic phrases) and what
  it is (clear structure, banned things, examples)

**Demo moment:** Open `sql_generation.txt` in the Tiri repo. Scroll
through it slowly, narrating each section. Then `git log -p` on
the file — show the actual history of how the prompt evolved.
This is a piece of *content*, with revisions, like marketing copy.

**Pairs with:** Module 05.

---

### 8a. The agent pattern: decomposition + IntentAgent (20 min)

First half of the densest module.

**Cover:**
- "What an agent is": prompt + LLM call + parser + typed result
- Why decomposition wins: each agent is unit-testable; failures
  localize; you can swap models per task
- IntentAgent walkthrough: prompt → JSON response → typed
  `IntentResult`
- Defensive JSON parsing (tolerate fences, never repair)
- Structured output / JSON mode as a concept

**Demo moment:** Live run of `IntentAgent.run()` against three
questions: an answerable one, an ambiguous one, a refused one.
Watch the JSON come back from the LLM and parse into typed
results. The agent's job is the plumbing; the LLM's job is the
classification.

**Pairs with:** Module 06, checkpoints 1–3.

---

### 8b. SQLAgent + validate-and-retry (20 min)

Second half. The pattern that generalizes.

**Cover:**
- Generate → validate → if invalid, append the validator's error
  to the conversation and retry
- Why this works: validators produce useful errors; modern LLMs
  apply them well
- Markdown fence stripping and other defensive output handling
- `CANNOT_ANSWER:` as an honest output, not a failure
- Structural enforcement: when a prompt's "don't say X" is not
  enough, scan the output and re-prompt

**Demo moment:** Manually break a fixture so the SQL agent's first
generation references a missing column. Watch attempt 1 fail
validation; watch attempt 2 — informed by the error — succeed.
Three lines of conversation history; one of the highest-leverage
patterns in the course.

**Pairs with:** Module 06, checkpoints 4–7.

---

### 9. First end-to-end answer (25 min)

The arc the entire course has been building toward.

**Cover:**
- The orchestration pattern: a method, not a framework
- Why `RoomConfig` reloads every request
- "Degrade, don't crash" — every step's failure is a structured
  error Turn
- The branch on intent: answer / clarify / refuse
- Conversation history (briefly)

**Demo moment:** Run `python scripts/ask.py sales-room "What was
total revenue by region?"` from a cold start. Watch each line of
output appear. Then ask "Why is revenue declining?" — watch the
agent refuse. Then ask "tell me about sales" — watch it ask for
clarification. **Three behaviors, one engine.** This is the
emotional payoff of the first 9 videos; let it breathe.

**Pairs with:** Module 07.

---

### 10. Streaming and the trust mechanism (18 min)

The trust-as-protocol arc.

**Cover:**
- REST in 60 seconds (skim if your audience already knows it)
- SSE: what it is, why it's enough for an agent (skip WebSockets)
- The events the engine emits, in order: status → intent → sql →
  result → synthesis → done
- **Why streaming the SQL early is the trust mechanism it is:**
  the user sees the query before the answer commits to it
- A brief note on auth (deferred)

**Demo moment:** Open a terminal. Run a streaming `curl -N` against
your agent. Watch the events arrive in real time. Pause on the
`sql` event — *this is when the user could challenge the query*.
Then let the synthesis arrive. The 2-second gap between SQL and
synthesis is the protocol's gift.

**Pairs with:** Module 08.

---

### 11a. Benchmarks as the spec (20 min)

The discipline module, first half.

**Cover:**
- Benchmarks vs unit tests: when each runs, what each tests
- SQL normalization: the rules, the pragmatism, the edge cases
- The two-tier match: `sql_match` OR `result_match`
- "100% or it's a bug" — what this rule means and what it doesn't
- Example engineering vs code engineering: where the bug usually
  lives

**Demo moment:** Run a benchmark suite that's intentionally at
80%. Walk through *one* failure live — read the generated SQL,
compare to expected, identify the missing example, add it to the
index, re-run. The failure-to-fix loop, in 3 minutes.

**Pairs with:** Module 09, checkpoints 1–6.

---

### 11b. Structural enforcement: the synthesis regex (15 min)

The discipline module, second half. The course's most important
single pattern.

**Cover:**
- The synthesis agent: question + SQL + result → plain English
- The causal-language ban: the prompt asks; the regex enforces
- Why "the prompt is enough" is not enough for a high-stakes
  system
- The re-prompt-on-violation pattern; fall back to mechanical if
  it persists
- Where else this pattern applies (everywhere you have a hard
  correctness rule)

**Demo moment:** Live re-prompt: feed the synthesis agent a
context that invites a causal claim. Watch the first response
include "caused by". Watch the regex catch it. Watch the second
attempt, prompted with the specific violation, produce clean
output. Show the test that asserts this works.

**Pairs with:** Module 09, checkpoint 7.

---

### 12. Composability and the long tail (20 min)

The "what's next" arc.

**Cover:**
- Multi-model routing: cheap for intent, smart for SQL. Why
  per-task model choice matters at scale.
- The `RouterLLMProvider` — same `LLMProvider` interface, dispatch
  inside
- MCP, recapped: server side and client side
- Brief tour of the other extensions you didn't build (planning,
  hypothesis mode, per-user UC, wildcards)
- "Extension beats rewrite" — the architecture's payoff

**Demo moment:** Set up the agent as an MCP server. Open Claude
Desktop. Show the `ask` tool appearing in the tool picker. Send
Claude a question that triggers the tool. Watch Claude call your
agent, get the answer, and weave it into its own response. *Your
data agent is now composable with any MCP-aware client.*

**Pairs with:** Module 10.

---

### 13. Capstone + close (10 min)

Wrap.

**Cover:**
- The four capstone options (real room, new extension, config
  translator, multi-provider study)
- The quality bar: runs, tested, honest, written up
- The five takeaways from the course (architecture is trust;
  benchmarks are spec; decomposition; refusal is a feature;
  extension beats rewrite)
- A note on what this course intentionally didn't teach (LLM
  internals, fine-tuning, RLHF, prompt-engineering folklore)
- A close: "go build something defensible"

**Demo moment:** None. This is the credits roll.

**Pairs with:** Module 11.

---

## Recommended viewing/build cadence

For a viewer working through this as a self-paced course:

| Week | Watch | Build (written module) |
|---|---|---|
| 1 | Videos 0, 1 (optional), 2 | Module 0 |
| 2 | Video 3 | Module 1 |
| 3 | Video 4 | Module 2 |
| 4 | Videos 5, 6 | Modules 3, 4 |
| 5 | Video 7 | Module 5 |
| 6 | Video 8a | Module 6 (checkpoints 1–3) |
| 7 | Video 8b | Module 6 (checkpoints 4–7) |
| 8 | Video 9 | Module 7 |
| 9 | Video 10 | Module 8 |
| 10 | Video 11a | Module 9 (checkpoints 1–6) |
| 11 | Video 11b | Module 9 (checkpoint 7) |
| 12 | Video 12 | Module 10 |
| 13–16 | Video 13 | Module 11 (capstone) |

About 4 months at ~4 hours/week. Faster for viewers who skip the
GenAI primer and have prior async-Python experience; slower for
viewers building serious capstones.

## Production notes

A few choices worth deciding up front:

**1. Speaker on camera or voiceover?** The vision and discipline
arcs (videos 0, 2, 9, 11b, 13) benefit from a face — they're
persuasive content. The mechanics arcs (3, 4, 5, 8a, 8b) can be
voiceover with screen recording. Mix is fine.

**2. Live coding vs prepared snippets?** Prepared snippets shown
in the editor. Run them live. **Don't type during the video** —
viewers don't watch you type, and you'll fat-finger something
once per episode. The exception: brief one-line edits during a
demo loop (a benchmark fix, a regex tweak) are fine if they're <30
seconds.

**3. Tiri repo or learner's parallel repo?** Show *both*. Tiri is
the canonical reference; the parallel repo is what the viewer
will build. Switch between them at natural beats — "here's Tiri's
version, here's the slimmer one you'll build."

**4. Music?** Quiet under the demo moments; off during exposition.
Standard YouTube/course conventions apply.

**5. Captions.** Required. Many of your viewers will watch in
contexts where audio is awkward (open offices, transit). Burn the
captions in.

**6. Code visibility.** 16pt minimum font for editor screen-shares;
larger if you can. A 12pt font that looks fine on your monitor is
illegible on a phone. Test every code clip at 720p before
shipping.

## What to film *first*

If you film one video to test the format, film **Video 9 (First
end-to-end answer)**. Reasons:

- It's the emotional payoff of the entire course. If you can't
  make this one land, the format isn't right yet.
- It's a single concrete demo with low setup cost — three
  questions through a working agent.
- It exercises every production skill the rest will need: screen
  share, a terminal demo, narration over running code, the
  pacing of a payoff beat.

After that, film **Video 4 (The provider boundary)**. It's the
test of whether you can sell an *architectural* idea in 20 minutes
without losing viewers — the hardest part of the course content.

The rest will be easier once those two work.

## Open questions

Things I'd want a producer's opinion on before shooting:

1. **One channel or two?** Modules 0–7 are foundation; Modules 8–10
   are surface and composition. They have different paces. Two
   playlists or one continuous series?

2. **The optional GenAI primer (Video 1).** Should this be a
   prerequisite, a "watch if you need it" callout, or pulled out
   into a separate standalone offering? Different audiences will
   want different things.

3. **Recording the labs anyway.** Should you also film "lab
   walkthroughs" — abbreviated 15-minute versions of each lab,
   for viewers who get stuck? Doubles the production work but
   doubles the support surface.

4. **Live cohort component.** Do you run a Discord/Slack alongside
   for live Q&A and capstone showcases, or stay fully self-paced?
   The course was designed as self-paced; cohort is an addition,
   not a substitution.
