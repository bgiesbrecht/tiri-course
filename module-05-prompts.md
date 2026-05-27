# Module 5 — Prompts as files

> **The rule:** Prompts are product surface, not code. They live in
> files. They have version history. They are not inline f-strings.

In this module you'll extract the prompts your agents will use into
text files, build a tiny loader that injects context at render time,
and write tests that pin both the structure (the template) and the
behavior (against fixture data). By the end your `agent/` directory
will have a `prompt_templates/` folder full of `.txt` files that you
can edit without re-running Python.

This module has more concepts and fewer checkpoints than the last
few. Prompts are short to write but consequential — most of the
module is about *what makes a good prompt*, not *how to load one*.

## Read

1. **`tiri/tiri/engine/prompt_templates/sql_generation.txt`** — read
   in full. This is the prompt the SQL agent uses. Notice:
   - It is one long structured document, not a single instruction.
   - Sections (`## Available tables`, `## Mandatory filters`, etc.)
     are visually separated and labelled. The model reads in order.
   - Placeholders look like `{table_schemas}` and `{question}`.
     These get filled in at request time from a `ContextPackage`.
   - There are *negative* instructions: "Return ONLY the SQL. No
     markdown fences. No explanation." Many prompts forget to forbid
     things; this one doesn't.

2. **`tiri/tiri/engine/prompt_templates/intent_classification.txt`**
   — also in full. Notice the difference in shape: this one asks the
   model to *classify* (choose between predefined paths), and the
   structure is built around making that choice clear.

3. **`tiri/tiri/engine/prompt_templates/synthesis.txt`** — skim. This
   is the one with the most opinion baked into the prompt: "never
   say 'caused by'". You'll see how the prompt-level guidance pairs
   with the post-generation regex scan (Module 6). Belt *and*
   suspenders.

4. **`tiri/tiri/engine/agents/sql_agent.py`**, lines 50–80 — see how
   the prompt is loaded once at module top (`_TEMPLATE =
   load_template("sql_generation.txt")`) and rendered per-request
   with `render(_TEMPLATE, ...)`. The loading happens once; the
   rendering happens many times.

## Concepts

### Why prompts can't be f-strings

The case for inline f-strings is "it's the same thing, just shorter":

```python
# Tempting. Don't.
prompt = f"""
You are a SQL expert. The user asked: {question}.
Tables: {tables}. Generate SQL.
"""
```

Four things break, all of them at once, the moment your prompt
matters:

1. **You can't iterate on the prompt without editing Python.** A
   prompt is a piece of product copy. Marketing teams rewrite product
   copy. Data scientists rewrite prompts. Forcing a Python file edit
   for every prompt tweak means the people who should be tuning the
   prompt can't.

2. **You can't diff a prompt change cleanly.** F-string changes show
   up mixed in with code changes. A reviewer can't see "what changed
   in the prompt" separately from "what changed in the function that
   uses it."

3. **You can't run a prompt through a tool.** Want to A/B test two
   prompt versions? Pipe a prompt through a token counter? Render a
   prompt with fixture data and inspect it? All trivial when prompts
   are files. Painful when they're embedded in code.

4. **You confuse two timelines.** Code changes are tied to deploys.
   Prompt changes can happen at any time, ideally without a deploy.
   Mixing them ties your "fix a typo in the prompt" to your release
   process — a friction that compounds.

There is no advantage to inline prompts beyond "fewer files." Make
the files.

### The structure of a working prompt

Most prompts that fail in production fail for the same reasons. The
ones that work tend to share a structure:

1. **Role and goal (one or two lines).** "You are X. Your job is Y."
   Direct. No flattery.

2. **Constraints up front.** "Return ONLY SQL. No markdown. No
   explanation." The model is more likely to obey constraints stated
   early.

3. **Context, in named sections.** `## Available tables`, `##
   Mandatory filters`, `## Examples`. The model finds information by
   scanning headers. Use them.

4. **Examples (few-shot), if any.** Two or three concrete (question,
   answer) pairs. More than a few wastes tokens; fewer than two often
   doesn't pin the format.

5. **The actual question, last.** Models attend more strongly to the
   end of the prompt. Put what you want answered there.

6. **Output format reminder, immediately before the model responds.**
   "Now generate the SQL. Return only the SQL." A short repeat at the
   end is worth more than a long preamble.

> **Concept: System prompt.** The first message in a conversation,
> with `role="system"`. It sets the model's identity, constraints,
> and rules for the whole exchange. Most LLM providers honor it
> strongly; some weight it as roughly "instructions you should not
> override later." Tiri puts the entire structured prompt template
> in the system message; the user message is usually empty or just
> "Proceed."
>
> **Concept: Few-shot examples.** Concrete (input, output) pairs
> included in the prompt to show the model what shape of answer you
> want. "Zero-shot" = no examples; "few-shot" = a handful. Few-shot
> is usually a noticeable quality boost for the cost of some tokens.

### Template variables and rendering

Tiri uses Python's built-in `str.format()` for rendering. That's
sufficient. Don't reach for Jinja2 or Mako unless you actually need
conditionals or loops in your templates — the moment you do, the
template stops being "the prompt the model sees" and starts being "a
program that prints the prompt."

The trade-off: with `str.format()` you can't have logic in the
template. So if you need conditional sections ("only include this if
the user has hypothesis mode on"), you do the conditional in Python
and pass `"(none)"` (or an empty string) as the value when the
section doesn't apply. This is the pattern Tiri uses everywhere.

Example from `sql_generation.txt`:

```
## External context (from MCP tools)
{mcp_context}
-- Lookup results from authorized external systems...
```

If MCP isn't configured, the rendering code passes `mcp_context=
"(none)"`. The section header still appears; the placeholder just
says "(none)". This is intentional — the model can tell the
difference between "no external context exists" and "this section
was suppressed." The former is honest; the latter is brittle.

### "Prompt engineering" — what it isn't

There is a cottage industry around prompt engineering — "use this
phrase to make models cite sources" / "this magic word improves
accuracy 12%." Most of it is local to a model version, falls apart
on the next release, and isn't the bottleneck in a real system.

The bottleneck in your system will be: **the prompt doesn't have the
right information.** The fix is almost always one of:

- More relevant context (better retrieval, Module 4)
- Smaller scope per agent (decomposition, Module 6)
- Validation outside the model (Module 2's `validate()` pattern)

These are architectural moves, not prompt tweaks. By the time you
are reordering words in your prompt to fix a quality issue, you have
usually skipped a step.

What does belong in prompt engineering, in this course's sense:

- Naming things clearly. (`## Mandatory filters` beats `## Filters`.)
- Forbidding things you don't want. (Banning markdown fences. Banning
  "caused by".)
- Providing two or three concrete examples for tricky cases.
- Asking for structured output (a specific JSON shape, a SQL string
  with no preamble) and providing exactly one example of that shape.

> **Concept: JSON mode / structured output.** Some LLM APIs let you
> demand a JSON-shaped response (OpenAI: `response_format={"type":
> "json_object"}`; Anthropic: tool use with a schema). The model is
> guided to produce valid JSON matching your schema. Useful for
> agents that need to return typed data. Tiri uses this for
> `IntentAgent`'s classification output. We'll explore it in
> Module 6.

## Lab

You'll extract a prompt for an `IntentAgent`-style classifier into a
file, write a loader, and test the result against fixture data. Four
checkpoints.

### Checkpoint 1 — Make the directory and the loader

In `~/my-agent/`:

```bash
mkdir -p agent/engine/prompt_templates tests/fixtures
touch agent/engine/__init__.py agent/engine/prompt_templates/__init__.py
touch agent/engine/prompts.py
```

`agent/engine/prompts.py`:

```python
"""Tiny prompt template loader and renderer.

Templates live in agent/engine/prompt_templates/*.txt.
Loaded once at import time; rendered with str.format() per request.
"""
from __future__ import annotations
from pathlib import Path

_TEMPLATE_DIR = Path(__file__).parent / "prompt_templates"


def load_template(name: str) -> str:
    """Load the template text from disk.

    Called once per template at module import time by the agents.
    Don't call this on the per-request path — read the file once
    and cache the string in a module-level constant.
    """
    path = _TEMPLATE_DIR / name
    if not path.exists():
        raise FileNotFoundError(
            f"prompt template not found: {path}. "
            f"Available templates: {sorted(p.name for p in _TEMPLATE_DIR.glob('*.txt'))}"
        )
    return path.read_text()


def render(template: str, **kwargs) -> str:
    """Render the template with kwargs.

    Missing keys raise KeyError — we don't silently substitute. A
    missing variable in production is a bug worth crashing on.
    """
    return template.format(**kwargs)
```

Two design decisions worth noticing:

1. **`load_template` reads from disk.** That looks slow, but it's
   called once per template at module import — typically four or
   five times for a whole agent system. The result is held in a
   module-level constant. Per-request cost is zero.

2. **`render` raises `KeyError` on missing keys.** Tempting to
   silently default to `""`. Don't. A missing variable in a prompt
   means the rendering code forgot to pass something the prompt
   depends on — that's a real bug. Letting it crash at render time
   is much cheaper than letting it succeed with a malformed prompt.

### Checkpoint 2 — Write the prompt

Create `agent/engine/prompt_templates/intent_classification.txt`.
This is a prompt for an agent that decides whether a user question
is answerable from data or needs clarification. Customize for your
domain (the manifesto from Module 0).

A starting point, modeled after Tiri:

```
You are classifying a user question for the <YOUR DOMAIN> agent.

The agent answers questions using the following data:
{table_descriptions}

The agent CANNOT answer questions that:
{refused_claim_types}

## Instructions
{text_instruction}

## Past examples of well-handled questions
{example_questions}

## Question
{question}

Classify the question into exactly one of:
- "answerable": the question can be answered with the available data
- "clarification_needed": the question is ambiguous; ask the user for
  more information before proceeding
- "refused": the question requires one of the refused claim types

Return your answer as JSON with exactly this shape:
{{
  "intent": "answerable" | "clarification_needed" | "refused",
  "reasoning": "<one sentence on why>",
  "clarification": "<the question to ask the user, if intent is clarification_needed; else empty string>"
}}

Return only the JSON. No markdown fences. No preamble.
```

A few things to notice:

- The `{{` and `}}` (doubled braces) — that's how `str.format()`
  escapes literal braces. Single `{` and `}` are placeholders;
  doubled ones survive into the output.
- The output schema is shown literally, including the field names and
  types. Models do much better when shown the exact shape, in the
  exact prose, that you want back.
- "Return only the JSON. No markdown fences. No preamble." — three
  short prohibitions at the end. The model has read everything else
  by now; what you forbid here sticks.

### Checkpoint 3 — A first render

Add a quick smoke test in `tests/test_prompts.py`:

```python
from agent.engine.prompts import load_template, render


def test_intent_template_loads():
    tpl = load_template("intent_classification.txt")
    assert "You are classifying" in tpl
    assert "{question}" in tpl     # placeholder still present before render


def test_intent_template_renders_with_full_context():
    tpl = load_template("intent_classification.txt")
    rendered = render(
        tpl,
        table_descriptions="- orders: customer orders, one row per order",
        refused_claim_types="- Causes of customer churn",
        text_instruction="(none)",
        example_questions="- Q: total revenue by region\n  A: SELECT region, SUM(amount)...",
        question="What was last quarter's revenue?",
    )
    assert "{question}" not in rendered   # all placeholders consumed
    assert "last quarter's revenue" in rendered
    assert "answerable" in rendered       # schema text survived


def test_render_raises_on_missing_variable():
    import pytest
    tpl = load_template("intent_classification.txt")
    with pytest.raises(KeyError, match="table_descriptions"):
        render(tpl, question="x")
```

Run:

```bash
pytest tests/test_prompts.py -v
```

Expected output:

```
tests/test_prompts.py::test_intent_template_loads PASSED
tests/test_prompts.py::test_intent_template_renders_with_full_context PASSED
tests/test_prompts.py::test_render_raises_on_missing_variable PASSED
```

These tests don't validate the *quality* of the prompt — only that
it loads and renders cleanly. Quality is verified by benchmarks
(Module 9). Right now we're just confirming the plumbing works.

### Checkpoint 4 — Inspect the rendered prompt

This is the single most useful debugging skill for prompt work: read
what the model actually sees.

Create `agent/engine/prompts.py` an interactive dump (or add it to a
notebook):

```python
# at the bottom of agent/engine/prompts.py, behind __main__
if __name__ == "__main__":
    tpl = load_template("intent_classification.txt")
    rendered = render(
        tpl,
        table_descriptions="- orders: customer orders, one row per order",
        refused_claim_types="- Causes of customer churn",
        text_instruction="(none)",
        example_questions=(
            "- Q: total revenue by region\n"
            "  A: SELECT region, SUM(amount) FROM orders GROUP BY region\n"
            "- Q: which customer ordered the most?\n"
            "  A: SELECT customer_id, COUNT(*) FROM orders "
            "GROUP BY customer_id ORDER BY 2 DESC LIMIT 1"
        ),
        question="What was last quarter's revenue?",
    )
    print(rendered)
```

Run:

```bash
python -m agent.engine.prompts
```

Read the output top to bottom. **Read it as the model would** — no
prior knowledge, just the text. Ask yourself:

- Are the section headers visually distinct?
- Does the schema example make the structure of "answerable" /
  "clarification_needed" / "refused" obvious?
- Is the question prominent at the end?
- Are the negative instructions ("Return only JSON...") still
  visible after all the context, or buried?

This kind of reading is half of prompt engineering. The other half
is letting the benchmark numbers (Module 9) tell you when your
intuition was wrong.

### Definition of done

- `agent/engine/prompts.py` defines `load_template` and `render`.
- `agent/engine/prompt_templates/intent_classification.txt` exists,
  customized for your domain.
- Three tests pass: loads, renders, raises on missing variable.
- You have run `python -m agent.engine.prompts` and read the output
  with the eyes of a model that has no prior context.
- Commit:
  ```bash
  git add . && git commit -m "module-05: prompt loader + intent template"
  ```

### Common pitfalls

1. **Literal `{` or `}` in the prompt text without doubling.** If
   you want a literal `{` in the output, write `{{`. If you forget,
   `str.format()` will raise an obscure `IndexError` or `KeyError`
   pointing at a placeholder you never defined.

2. **You build the prompt as a Python string for "convenience," then
   put it in a file later.** This always becomes "always later."
   Move it to a file now, even if the file is one line. The friction
   of moving an established prompt later is higher than the friction
   of starting in a file.

3. **You load the template inside the agent's request handler.**
   That's an unnecessary disk read per request. Load at module top
   (`_TEMPLATE = load_template("...")`) and re-use. The cost
   difference is small but the pattern is the right one.

4. **Your prompt is one long unbroken paragraph.** Models perform
   measurably worse on unstructured walls of text. Use sections.
   Use blank lines. Use bullet lists. Treat the prompt as a document
   the model reads, because it is.

5. **You use Jinja2 to handle a single conditional.** Resist. Pass
   `"(none)"` as the value when the section doesn't apply. The model
   handles "(none)" cleanly; you handle one less dependency.

## Stretch

1. **Add a `synthesis.txt` template** for an agent that summarizes
   query results into plain English. Include a section listing
   *banned phrases* ("caused by", "because of", "led to") so the
   model is warned not to use them. (Module 6 adds the regex scan
   that enforces the ban after the model responds — belt and
   suspenders.)

2. **Render fixtures.** Build a `tests/fixtures/intent_examples.py`
   with five or six concrete `(rendered_prompt, expected_intent)`
   pairs. You can't run them against a real model in tests, but you
   can pin the *rendered output* with a snapshot test:
   `assert rendered == expected_text`. This catches accidental
   template changes that would change every prompt the system
   sends.

3. **(Hard)** Build a small CLI: `python -m agent.engine.prompts
   --template intent_classification.txt --question "..."`. It
   renders the template with reasonable defaults for the other
   variables and prints the result. This becomes a daily-use tool
   when you're tuning prompts.

## Reflection

1. The rule "render raises on missing variables" was a deliberate
   choice in checkpoint 1. The alternative — silently substitute
   empty string — would prevent a class of crashes in production.
   Why is "crash on missing variable" the better default?
   (Hint: think about a placeholder that *gets added later* and isn't
   passed by the rendering code. Silent default = bad prompt goes to
   the model. Crash = developer notices.)

2. Your prompt template uses `str.format()`. Tiri uses the same. Why
   not Jinja2, which is a superset? What is the smallest piece of
   logic you'd need in a template before the choice flips?

3. Compare the structure of your `intent_classification.txt` to
   Tiri's. What sections are in Tiri's that aren't in yours, and why
   does Tiri have them? Is there one you could add to your prompt
   that would let your agent do something it currently can't?
