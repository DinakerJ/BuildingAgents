# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

UdaPlay — a Udacity capstone building an AI game-research agent: RAG over a local 15-game corpus,
Tavily web-search fallback, a stateful ReAct agent, and an evaluation harness.

- **Python + Jupyter notebooks only.** No CLI, no server, no app to run. The notebooks *are* the
  deliverable.
- **`project/` is the submission artifact** — only project files go in it. The repo root is the
  workspace (docs, trackers, dev manifest).
- **Graded deliverable filenames** are `Udaplay_01_solution_project.ipynb` and
  `Udaplay_02_solution_project.ipynb`. They live *beside* the starters in `project/starter/` so the
  relative `lib.` imports and the `"games"` path keep working.
- The scenario, rubric, and the user's own instructions live in [Agent.md](Agent.md). Read it; do
  not restate it here.

## Operating contract — tutor mode

**This is the most important section in this file.** The user is presenting this project to a panel
and must be able to explain every line. Producing correct code the user cannot defend is a failure,
not a success.

- **Explain the concept before writing the code.** Never lead with a code block.
- **Be brief.** Explanations are short and to the point — a few lines, not essays. Expand only when
  the user asks to go deeper ("explain in detail", "break it down"). Brevity does not cancel the
  teach-back requirement below: short explanation, then the question.
- **Every line of code written into `project/` gets explained, always.** This overrides brevity —
  the two do not conflict, because brevity governs prose, not code coverage. The user is a beginner
  and has to defend every line to a panel, so a summary of what a cell *does* is not enough. Walk
  the actual code: each new language construct, each library call, each design choice. Group into
  small labelled blocks rather than one wall of text. Never skip a snippet because it "just" does
  setup or checking.
- **Prefer less code over more.** Every line written is a line the user must defend. If a shorter
  version would do, offer it.
- **When asked to explain**, use short broken-down sections under small headings, in simple
  non-technical English. Answer the specific confusion first. A small table beats a paragraph when
  comparing things.
- **Write plainly.** No AI-flavoured filler — *leverage, robust, seamless, delve, comprehensive,
  landscape, unlock, pivotal, crucial, testament, journey*, or openers like *"it's important to
  note"*. Plain verbs, concrete nouns. This applies to notebook markdown and code comments too,
  since the user has to read it aloud to a panel.
- **One small unit per turn** — one tool, one step, one cell. Then stop.
- **After each unit, ask a comprehension question and wait for the answer.** Do not proceed on
  silence or on "ok".
- **Never chain multiple notebook cells or multiple files in a single turn.** Even when the next
  three steps are obvious to you.
- **Never silently fix anything.** Bugs get the protocol below, not a quiet patch.
- **Never auto-advance a phase.** Ask before recording a gate in [PROGRESS.md](PROGRESS.md).

### Bug protocol

Bugs are fixed **when the sprint reaches them**, never pre-emptively. When one is hit — whether it's
a catalogued defect from [BUGS.md](BUGS.md) or something new — stop and walk through it in this
order, *before* writing the fix:

1. **Highlight it.** Name the bug and point at the exact `file:line`. Don't fold it into a larger
   explanation of something else.
2. **Why it's a bug.** The underlying mechanic, not just the symptom — what the code assumes that
   isn't true. The user should be able to predict the failure from the mechanic afterwards.
3. **Impact.** Blast radius and stakes: does it fail loudly or silently? What's blocked? Is it a
   rubric requirement? What will they waste time on if it goes unfixed? **State explicitly whether
   it fails loudly or silently** — the silent ones (B2, B5) are the dangerous ones and the user
   needs to internalise the difference.
4. **Then fix it**, and confirm the fix by observing the symptom disappear.
5. **Log it** in the `BUGS.md` fix ledger and `PROGRESS.md` §3 with the rationale.

For the catalogued bugs, `BUGS.md` already has all of steps 1–3 written up plus a panel answer —
walk the user through that entry rather than paraphrasing it from scratch. If a *new* bug turns up,
add an entry in the same shape.

Two entries (N1, N2) are **not bugs** — they look like defects but aren't. Don't "fix" them, and
don't let the user claim them as fixes to a panel.

### The phase gate

> A phase is complete only when **Built ∧ Tested ∧ Explained-back-by-the-user**.

Two of three is not complete. Working code that the user cannot explain does not pass. The **only**
record of gate state is `PROGRESS.md` — and only the user marks *Explained*.

### Do not

- Refactor `lib/` for style, naming, or taste. Touch it only for the defects in `BUGS.md`.
- Add abstractions the rubric does not ask for.
- Add a test framework — see *Testing* below.
- Ghostwrite the user's prose in `project/README.md` beyond a skeleton. It is their submission.

## Commands

### Environment

The checked-in `.venv` is **CPython 3.14 with only ipykernel installed** — no project dependency is
importable today. ChromaDB 1.0.x is unlikely to have 3.14 wheels, and the project targets 3.11.

```bash
uv venv --python 3.11
uv pip install -r requirements.txt          # root manifest; there is none inside project/
uv run python -m ipykernel install --user --name udaplay --display-name "UdaPlay (3.11)"
```

### Running the notebooks

**Notebooks must run with cwd = `project/starter/`.** They do `from lib.agents import Agent` and
read the relative path `"games"`. Launched from anywhere else, the imports die.

```bash
cd project/starter && jupyter lab
```

Both notebooks open with a `pysqlite3` shim that swaps into `sys.modules['sqlite3']` — ChromaDB
needs a newer SQLite than some hosts ship. Leave it alone.

### Testing

There is **no pytest, no CI, no `tests/` directory**, and none should be added. "Testing" means two
things here:

1. **Restart-kernel + run-all.** This is the only trustworthy check, because it is exactly what a
   grader does. A notebook that only works incrementally is not done.
2. **The three canonical smoke queries** (from `project/starter/README.md`), each proving something
   different:

   | Query | What it proves |
   | --- | --- |
   | "When was Pokémon Gold and Silver released?" | Plain RAG hit — the fact is in the corpus |
   | "Which one was the first 3D platformer Mario game?" | RAG + reasoning across descriptions |
   | "Was Mortal Kombat X released for PlayStation 5?" | **The fallback path.** Not in the 15-game corpus, so retrieval must miss, `evaluate_retrieval` must reject it, and `game_web_search` must fire |

   The third is the integration test. It is the only query that exercises the fallback edge — if it
   answers from RAG, something is wrong.

3. **The eval harness** is `lib/evaluation.py::AgentEvaluator`: `evaluate_final_response`
   (LLM-as-judge with a heuristic fallback), `evaluate_single_step` (tool-selection correctness),
   and `evaluate_trajectory` (walks `run.snapshots`, counts steps and tools, estimates cost).

## Architecture of `project/starter/lib/`

`lib/` is a hand-rolled mini-LangChain provided by the course (~1,900 lines, pre-written). Two
mental models span multiple files and cannot be learned from any one of them.

### Model A — the agent *is* a state machine

`agents.py` is **not** a hand-rolled ReAct loop. It is `StateMachine[AgentState]` wired as:

```
entry → message_prep → llm_processor → (tool_executor ⇄ llm_processor | termination)
```

The loop-back edge `tool_executor → llm_processor` *is* the ReAct cycle. The branch is
`check_tool_calls` at [agents.py:133-139](project/starter/lib/agents.py#L133-L139): tool calls
present → `tool_executor`, otherwise → `termination`.

The non-obvious coupling is in [state_machine.py:49-58](project/starter/lib/state_machine.py#L49-L58):

```python
expected_fields = get_type_hints(state_schema)
updated = {**state}                      # prior state carried forward
for field, value in result.items():
    if field in expected_fields:         # unknown keys SILENTLY dropped
        updated[field] = value
```

So the schema controls what a step may **change**, not what the state may **contain**. Reading
`agents.py` alone will never explain why a returned key had no effect. This mechanic is behind both
non-bugs in `BUGS.md` (N1, N2) — read those before touching `AgentState`.

`StateMachine.run` records a `Run` containing one `Snapshot` (uuid + timestamp + deep-copied state)
per step. That snapshot log is what makes trajectory evaluation possible.

### Model B — the data path

```
documents.py  (Document, Corpus)
      ↓
vector_db.py  (VectorStoreManager owns the chroma client + OpenAI embedding fn;
               VectorStore wraps one collection)
      ↓
      ├── rag.py            retrieve → augment → generate
      └── memory.py         LongTermMemory (same machinery, storing MemoryFragments)
```

`Corpus.to_dict()` produces the `contents` / `metadatas` / `ids` triple that Chroma's `add()`
expects. Chroma returns **columnar** results (parallel arrays keyed by field), not row records —
`query()` wraps them in one extra list level (one per query text), `get()` does not.

### Supporting cast

- **`messages.py`** — Pydantic `SystemMessage` / `UserMessage` / `AIMessage` / `ToolMessage`, plus
  `TokenUsage`. `AIMessage` carries `tool_calls` and `token_usage`.
- **`tooling.py`** — the `@tool` decorator reflects a function's type hints and docstring into an
  OpenAI function-calling schema. **The docstring becomes the tool description the model sees**
  ([tooling.py:24](project/starter/lib/tooling.py#L24)), so a wording change is a behaviour change,
  not a comment change. Handles `Literal`→enum, `Optional[T]`, `list`, `dict`, primitives.
- **`llm.py`** — thin OpenAI wrapper. Default `gpt-4o-mini`, temperature 0.0. Adds
  `tools` + `tool_choice: "auto"` when tools are registered; `response_format=SomeModel` routes to
  `client.beta.chat.completions.parse` for structured output.
- **`memory.py`** — `ShortTermMemory` (session → list of `Run`s, with a protected `"default"`
  session) vs `LongTermMemory` (vector-backed `MemoryFragment`s with namespace/owner/timestamp
  filtering).
- **`loaders.py` / `parsers.py`** — the PDF ingestion path. **Unused by UdaPlay.** Don't spend time
  there. (`pdfplumber` is still required, because `vector_db.py` imports `loaders`.)

### One query, end to end

1. `Agent.invoke(query, session_id)` → `ShortTermMemory.create_session(session_id)`
2. `get_last_object(session_id)` → the previous `Run`'s final state supplies `messages`
   — **this is what makes multi-turn context work** for the rubric
3. `initial_state` is built and the machine runs
4. `message_prep` prepends the `SystemMessage` (first turn only) and appends the `UserMessage`
5. `llm_processor` calls the LLM with the tool schemas; returns an `AIMessage`
6. If it requested tools → `tool_executor` dispatches by name to the matching `@tool`, appends
   `ToolMessage`s, and loops back to step 5
7. Otherwise → `termination`; the `Run` is appended to `ShortTermMemory` and returned

## Landmines

Code defects are catalogued in **[BUGS.md](BUGS.md)** (B1–B6 real, N1–N2 look-like-bugs-but-aren't)
with symptom, cause, mechanic, fix, and a panel answer for each. Don't duplicate that analysis here;
don't pre-apply the fixes.

Environment and state gotchas (current as of P1 gated, 2026-09-14):

- **cwd must be `project/starter/`.** The single most common cause of "it worked yesterday".
- **Environment is built.** Python 3.11.15 venv, all dependencies installed, kernel registered as
  **UdaPlay (3.11)**. `openai` is 3.10.0 and `chromadb` is 1.5.9 — both far above the `Agent.md`
  pins, both verified against the APIs `lib/` needs.
- **Notebook 01 is done through the ingest.** Client, embedding function, collection, ingest and a
  persistence check are all written and have real outputs. Notebook 02 is still entirely `# TODO`.
- **The store exists on disk** at `project/starter/chromadb/`, collection `udaplay`, 15 documents,
  embedded with `text-embedding-3-small`. Gitignored.
- **The ingest deviates from the starter, deliberately.** The sentence template adds `Genre` and
  `Publisher`, and it uses `upsert` rather than `add`. Both logged in `PROGRESS.md` §3 — don't
  "restore" them.
- **`CHROMA_OPENAI_API_KEY` is not needed.** chromadb 1.5.9 switches to `OPENAI_API_KEY` when that
  is set, and the OpenAI SDK passes `OPENAI_BASE_URL` through to Chroma's embedding client. The
  starter README's request for it is stale. See B3, which is **withdrawn**.

## Conventions

- Work in `*_solution_*.ipynb`. Leave the `*_starter_*.ipynb` files pristine as reference.
- **Keep notebook outputs committed.** The rubric grades visible reasoning, tool usage, and cited
  answers. Scrub any key that leaks into a traceback before committing.
- Every `lib/` edit gets logged in the `BUGS.md` fix ledger *and* `PROGRESS.md` §3 with its
  rationale. That log is panel ammunition — it doesn't get written retroactively.
- `requirements.txt` lives at the root only, by choice. At P10, `project/README.md` must therefore
  spell out its dependencies as text.
- `.env` is gitignored and stays that way.
