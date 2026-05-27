# Module 0 — Frame

> **The rule:** Vision is the tiebreaker. When two technically valid
> choices conflict, the system's purpose decides.

This module has no code. It has one job: get you to internalize what
Tiri refuses to be, so the architecture in later modules stops feeling
arbitrary. Skipping this module is the most common reason a learner
later asks "why didn't they just use a single big prompt?" — and then
ships an agent that bluffs.

Before reading: open the README and study the "big picture" diagram.
You don't need to understand every box. You need a mental map of the
shape of what we're building so the rest of this module lands.

> **Concept: LLM (Large Language Model).** A neural network that
> takes a chunk of text (the "prompt") and produces more text. GPT-4,
> Claude, Llama. You send a list of messages; you get a string back.
> The model has *no memory between calls* — anything it should know
> must be in the prompt.
>
> **Concept: Agent vs chatbot.** A chatbot replies. An agent *does
> things*: it inspects context, decides what to do next, calls a tool
> (a database, an API, another agent), looks at the result, and may
> take another step. In this course, an "agent" is a small unit —
> one prompt, one job, one structured return value. A pipeline of
> several agents working in sequence is what produces an answer.

## Read

In order, with what to look for in each:

1. **`tiri/docs/vision.md`** — the whole thing. Read it twice. The
   first pass tells you what Tiri does. The second pass tells you what
   it refuses to do, and why. Underline the sentence "Tiri is a
   witness, not an analyst." Most design decisions in later modules
   trace back to it.

2. **`tiri/CLAUDE.md`**, sections "What you are building" and "Key
   design rules" — skim. You'll come back to "Key design rules" in
   every module. For now, just know it exists.

3. **`tiri/docs/concept_map.md`** — optional. Skip unless you've used
   Databricks Genie before. If you have, this map tells you precisely
   where Tiri stops looking like Genie and starts looking different.

## Concepts

### Retrieval vs reasoning

A retrieval question has one answer in the data and one query that
gets it: *"What was revenue last quarter?"* A reasoning question
requires identifying the right tables, understanding contextual terms
("targets", "this quarter"), running several queries, and synthesizing
a coherent answer: *"How is the team performing against targets, and
where is the biggest gap?"*

Most agentic-LLM tutorials treat these the same. They aren't. The
second one is where confident-sounding wrong answers happen, because
the model has to make assumptions to produce *any* answer at all.
Tiri's architecture is shaped by the second question, not the first.

### The witness, not the analyst

A witness tells you what they observed, completely and precisely. They
don't speculate about causes. They don't recommend verdicts.

This sounds limiting. It is. The limitation is the product.

The audiences Tiri is designed for — congressional staffers, CEOs
making capital decisions, managers reporting on performance — *cannot
afford* an answer that sounds confident but rests on an unstated
causal assumption. The cost of getting it wrong, in those contexts,
is too high. So Tiri refuses to make the claim at all, and instead
shows the data and points at the gap.

The trick is that this constraint, applied honestly, makes the system
**more** useful in those settings, not less. A staffer who can say "the
data shows X, and the data does not support concluding why" has a
defensible position. A staffer who repeats a fluent causal claim from
an AI in testimony does not.

### Why fluent confidence is dangerous

LLMs are trained to produce coherent, confident-sounding text. That is
the default. To get the opposite behavior — explicit uncertainty,
visible evidence, "I don't know" when that is the true answer — you
have to fight the default at every layer: prompts, post-processing,
agent boundaries, output schemas, and yes, even regex scans for
banned causal language.

Each of those layers shows up in a later module. They're not paranoia.
They're how you make the system safe for the audience it serves.

### The cost side

This is not free. Building an agent this way means:

- More LLM calls per question (planning, intent, SQL, synthesis,
  sometimes hypothesis), not one.
- More engineering work to specify what the system refuses to claim.
- More benchmarks, more tests, more provider abstractions.
- A slower, more expensive system on a per-question basis.

The trade you're making: defensibility over delight, correctness over
speed. If your audience can tolerate a fluent guess, you don't need
this architecture. If they can't, the cost is the price of admission.

## Lab

This module's lab is conceptual. It will feel slight after later
modules' coding labs. Resist the urge to skip it — the document you
produce here is the spec you'll measure your capstone against.

Five checkpoints. Don't move past one until it's complete.

### Checkpoint 1 — Set up your environment

Pick your operating mode for the course. You'll keep this choice
through Module 6 or so; you can switch later.

**Local mode (no Databricks workspace required):**

```bash
git clone <Tiri repo URL> ~/tiri-reference
cd ~/tiri-reference
pip install -e ".[dev]"
cp .env.example .env.local
# Edit .env.local: set LLM_PROVIDER=openai or anthropic
#                  set OPENAI_API_KEY=... or ANTHROPIC_API_KEY=...
pytest tests/unit/
```

Expected output (last line):

```
======= 467 passed, 3 skipped in 3.12s =======
```

The 3 skipped are Databricks integration tests. That's fine.

**Databricks mode (full integration path):**

```bash
# Same clone + install as above
cp tiri.toml.example tiri.toml
# Edit tiri.toml: set databricks_host, llm endpoints, warehouse_id
pytest tests/unit/
```

Same expected output. If you want to also exercise the live
workspace path, set `INTEGRATION_TESTS=true` and rerun — but that's
optional for the course.

**Hybrid mode (recommended if you have access):** use local for
Modules 1–6, Databricks for 7–10 and capstone. The provider
abstraction makes this clean — your code doesn't change, only the
config.

Now create your own working directory *next to* the reference (not
inside it):

```bash
mkdir ~/my-agent && cd ~/my-agent
git init
python -m venv .venv && source .venv/bin/activate
echo ".venv/" >> .gitignore
echo ".env.local" >> .gitignore
git add .gitignore && git commit -m "module-00: scaffold"
```

Everything you build in this course goes in `~/my-agent`. The Tiri
repo stays untouched — it's a reference, not a fork target.

### Checkpoint 2 — Pick your domain

In your working directory, create `refusal-manifesto.md` and fill in
the **Audience** and **Question** sections.

```markdown
# Refusal manifesto for <agent name>

## Audience

<Who, specifically, will use this agent? Be concrete.>

## Question

<The single question this agent excels at. Pick ONE.>
```

Be specific. Here are examples by domain to anchor on:

| Domain | Bad audience | Good audience |
|---|---|---|
| Sales | "Sales leaders" | "Account executives preparing for a renewal conversation with a single named customer" |
| Engineering | "Engineering managers" | "Engineering managers preparing a quarterly headcount justification to their VP" |
| Public sector | "Government staff" | "Congressional staffers drafting testimony for a committee hearing" |

The single question should be answerable with the data you have
access to (or could plausibly get). If you find yourself writing
three questions, pick one — you can always grow the agent later, but
trying to make Module 0 cover three questions is what burns the
following 11 modules.

### Checkpoint 3 — Name what the agent refuses to claim

Add to `refusal-manifesto.md`:

```markdown
## Refused claims

The agent will not make the following kinds of claim, regardless of
how the user phrases the question:

1. <Claim type 1>
   - Why this matters: <one sentence on the liability if we made
     this claim wrongly>
2. <Claim type 2>
   - Why this matters: <same>
3. <Claim type 3>
   - Why this matters: <same>
```

A real example, from a hypothetical renewal-conversation agent:

```markdown
## Refused claims

1. Why a customer is at risk of churning. The agent can show usage
   trends and contract status. It will not name a root cause.
   - Why this matters: an AE who repeats an AI-generated reason in
     a customer call invites the customer to dispute that reason —
     a much harder conversation than presenting trends and asking.

2. Whether a customer "should" be approached for an upsell. The
   agent can show consumption headroom and tier-fit data. It will
   not produce a recommendation.
   - Why this matters: recommendations are an AE's job. An agent
     that crosses into recommendation territory will eventually
     produce a bad one, and the AE who acted on it bears the cost.

3. Predictions about future revenue. The agent can show historical
   trajectory; it will not extrapolate.
   - Why this matters: extrapolation across a customer's
     organizational changes is a quant-finance discipline. Doing it
     casually with an LLM is how you lose a deal on a bad forecast.
```

Notice the pattern: each refusal is a concrete claim type, not a
vibe. "I won't make causal claims" is too abstract. "I won't say
*why* a customer is churning" is testable.

### Checkpoint 4 — Define what "I don't know" looks like

Add to `refusal-manifesto.md`:

```markdown
## When the agent says "I don't know"

The agent has at least three ways of declining to answer. Use the
right one for the right situation.

- **The data is insufficient**: <example>
- **The question is outside scope**: <example>
- **Two sources disagree**: <example>
```

The exact text the agent produces in each case is what matters. From
the same renewal-conversation example:

```markdown
- **The data is insufficient**: User asks "did renewal probability
  drop after the last QBR?" but the agent has no signal for
  customer sentiment. Response: "I can't determine the impact of
  the QBR from the data available — usage stayed flat in the four
  weeks after, but I have no engagement signal to attribute that
  to. To answer this, you'd need data from <X>."

- **The question is outside scope**: User asks "what features
  should we ship to retain this customer?" Response: "That's a
  product roadmap question, not a data question I can answer. I
  can show you which features this customer uses most heavily, if
  that helps."

- **Two sources disagree**: SFDC says ACV is $X, the consumption
  report says burn rate implies $Y. Response: "Two sources disagree
  about this customer's annual contract value: SFDC says $X (last
  updated <date>), consumption data implies $Y. I'm showing both;
  I can't resolve which is correct without judgment."
```

The point of these scripts: when the agent is unsure, it should
produce something *useful*, not just throw up its hands. Pointing
the user at the next data source, or at the right human to ask, is
part of the job.

### Checkpoint 5 — Name the unforgivable failure mode

Add to `refusal-manifesto.md`:

```markdown
## The unforgivable failure mode

One sentence describing the specific failure this agent must never
have. If you read this back, it should make immediate sense why the
refusals above exist.
```

Tiri's:

> A non-technical user repeats an unstated causal claim from the
> system in a high-stakes setting (testimony, capital allocation,
> personnel decision), is challenged on it, and has no defense.

Yours should be specific to your audience. "The agent gives a wrong
answer" is too broad. "The user surfaces an AI hallucination to their
CEO and damages their own credibility" is specific.

### Definition of done

- `~/my-agent/refusal-manifesto.md` exists and is committed.
- All five sections are filled in (audience, question, refused
  claims, "I don't know" scripts, unforgivable failure mode).
- A friend or colleague can read it in under five minutes and tell
  you back, in their own words, what the agent refuses to do and
  why. If they can't, the document is too abstract — revise it.
- Run:
  ```bash
  cd ~/my-agent
  git log --oneline
  ```
  You should see at least one commit referencing the manifesto.

### Common pitfalls

1. **Three audiences instead of one.** You'll be tempted to widen the
   audience to "all the people who might use this." Don't. A useful
   agent has a sharp audience; a useless one has all of them.

2. **Refusing too much.** If your refusal list rules out every
   interesting answer, the agent is useless. Refusals should
   eliminate *liability*, not *value*. If you can't think of a
   single specific, simpler claim the agent still *can* make in
   place of a refusal, the refusal is too aggressive.

3. **Vibes-level refusals.** "The agent won't bluff" isn't a
   refusal; it's an aspiration. A refusal names a *kind of claim*
   the agent will not make under any phrasing — concrete enough
   that you could write a test against it.

4. **Writing the manifesto in the abstract.** If you can't picture
   the actual user with the actual question in their actual context
   (a hearing room, a customer call, a board meeting), keep
   thinking about Checkpoint 2 before writing the rest. The whole
   document collapses without a real audience in mind.

## Stretch

Read `tiri/docs/concept_map.md` in full, even if you haven't used
Genie. The mapping table is a useful inoculation against framework
thinking — it shows you that the *names* of the parts (snippets,
rooms, intents) are convention, and the *responsibilities* are what
matter.

## Reflection

Before moving to Module 1, sit with these for a few minutes:

1. The rule "Tiri does not make causal claims" feels conservative. Can
   you think of a domain where that rule would be too conservative —
   i.e., would the agent be useless without making at least some
   causal claims? What would you do differently in that domain, while
   still earning your audience's trust?

2. Your refusal manifesto names three claims your agent will not make.
   For each, what is the simpler, more limited claim it *can* make
   that still has value? (If there isn't one, your refusal list might
   be too aggressive.)

3. If a user asks your agent a question that requires it to make one
   of the refused claims, what does the response look like? Is it
   silence? An apology? A redirection? Sketch the actual text.
