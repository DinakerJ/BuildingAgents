# PROGRESS.md — UdaPlay 5-day sprint

Plan and gate ledger. This is the single record of what is done. Nothing is complete unless it is
marked complete here.

## §0 How to use this file

**The gate rule:**

> A phase is complete only when **Built ∧ Tested ∧ Explained**.

- **Built** — the code exists and runs.
- **Tested** — the stated test passes on a *restarted kernel*, not incrementally.
- **Explained** — you answered the phase's `Understand` questions aloud, unprompted, without
  reading from the screen. **Only you mark this one.** Claude may not.

Two of three is not complete. Do not start the next phase.

**Status legend:** `[ ]` not started · `[~] ` in progress · `[B]` built · `[BT]` built + tested ·
`[x]` **GATED** (all three)

Budget: **3–4 focused hours/day**, ~18.5h total. The sprint has slack in it deliberately — a plan
that assumes every phase lands first time will fail its own gates on Day 2.

---

## §1 Sprint map

| Day | Phase | Hrs | Exit artifact | Status |
| --- | --- | --- | --- | --- |
| 1 | P0 Environment & recon | 1.5 | 3.11 venv, deps installed, preflight cell green | `[ ]` |
| 1 | P1 Persistent Chroma store + ingest | 2.0 | `udaplay` collection persisted on disk | `[ ]` |
| 2 | P2 Semantic search → finish Part 1 | 1.5 | Notebook 01 restart-run-all clean | `[ ]` |
| 2 | P3 `lib/` deep dive | 2.0 | You can explain how `@tool` becomes an OpenAI schema | `[ ]` |
| 3 | P4 The three tools | 2.5 | `retrieve_game`, `evaluate_retrieval`, `game_web_search` | `[ ]` |
| 3 | P5 First working agent | 1.5 | 3 smoke queries answered, incl. Tavily fallback | `[ ]` |
| 4 | P6 Stateful agent + state machine | 2.0 | Multi-turn session demonstrably remembers context | `[ ]` |
| 4 | P7 Long-term memory persistence | 1.5 | Web-search findings survive a kernel restart | `[ ]` |
| 5 | P8 Evaluation | 1.5 | AgentEvaluator: final-response, single-step, trajectory | `[ ]` |
| 5 | P9 Stand-out features | 1.0 | Structured JSON + citations; extended dataset | `[ ]` |
| 5 | P10 Polish & submit | 1.5 | `project/README.md` filled; both notebooks clean | `[ ]` |

Day totals: **D1** 3.5h · **D2** 3.5h · **D3** 4.0h · **D4** 3.5h · **D5** 4.0h

---

## §2 Phase blocks

### P0 — Environment & recon · Day 1 · 1.5h · Status: `[ ]`

**Goal.** A working Python 3.11 environment where every project dependency imports and both API
credentials are verified live.

**Concepts**
- Why 3.11 and not the 3.14 venv currently checked in (wheel availability for native deps).
- What `.env` + `python-dotenv` actually do, and why `OPENAI_BASE_URL` is read implicitly by the
  OpenAI SDK but not by ChromaDB's embedding client (see `BUGS.md` B3).
- Why notebook cwd matters: relative `lib.` imports and the `"games"` path.

**Build**
1. `uv venv --python 3.11`, then `uv pip install -r requirements.txt`.
2. Register the kernel: `python -m ipykernel install --user --name udaplay`.
3. Extend `.gitignore` with `**/chromadb/`, `.ipynb_checkpoints/`, `__pycache__/`.
4. Copy `Udaplay_01_starter_project.ipynb` → `Udaplay_01_solution_project.ipynb` (and the same for
   02). All work happens in the solution copies; starters stay pristine.
5. Write a **preflight cell** in notebook 01: load `.env`, assert the three keys are present, one
   `chat.completions` round-trip, one Tavily search, one embedding call, `chromadb.__version__`.

**Test.** The preflight cell runs green end-to-end on a fresh kernel, launched from
`project/starter/`.

**Understand**
1. Why does chat work through your gateway "for free" while embeddings might not?
2. What exactly breaks if you launch Jupyter from the repo root instead of `project/starter/`?
3. What is in `requirements.txt` that `Agent.md` didn't list, and why is it needed?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**
> _If the embedding call in the preflight succeeds unmodified, B3 does not apply — amend `BUGS.md`._

---

### P1 — Persistent Chroma store + ingest 15 games · Day 1 · 2.0h · Status: `[ ]`

**Goal.** Load, process, and embed the 15 game JSON files into a **persistent** ChromaDB collection.
(Rubric: *"The processed data is added to a persistent vector database with appropriate
embeddings."*)

**Concepts**
- Embeddings as semantic coordinates: why `[Platform] Name (Year) - Description` is a better
  embedding target than raw JSON.
- Persistent vs ephemeral Chroma clients, and why the rubric demands the former (`BUGS.md` B2).
- What a collection's *embedding function* is, and when it runs (on `add`, and on `query` text).
- Metadata vs document content: what belongs in each, and what you can filter on later.

**Build**
1. Read the 15 files in `games/`; inspect the schema.
2. Hit and fix **B1** (`create_store` UnboundLocalError), **B2** (persistent client), **B3**
   (embedding `api_base`) — in that order, as each surfaces.
3. Create the persistent collection (`path="chromadb"`).
4. Wire up **notebook 01 cell 13** — the ingest loop is *already written*; it needs the `collection`
   defined above it. Don't rewrite it.

**Test.** Ingest, then restart the kernel, reconnect **without** re-ingesting, and confirm
`collection.count() == 15`. That restart is the whole point — it's what proves persistence.

**Understand**
1. Why does `create_store(force=True)` exist at all, and when would you want it?
2. If you changed the content template, what would you have to redo, and why?
3. Where does the embedding actually get computed — your machine or the API?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P2 — Semantic search → finish Part 1 · Day 2 · 1.5h · Status: `[ ]`

**Goal.** Demonstrate semantic search over the collection and close out Part 1.

**Concepts**
- Cosine distance vs similarity, and why "distance ≈ 0.3" means nothing without calibration.
- Columnar results: `query()` returns `{ids, documents, metadatas, distances}` as parallel arrays
  wrapped one level deep (one per query text); `get()` returns them unwrapped and has **no**
  distances (`BUGS.md` B4).
- Why `n_results` is a recall/precision dial, and what it costs you downstream in prompt tokens.

**Build**
1. Query the store with 3–4 natural-language questions; print documents alongside distances.
2. Hit and fix **B4** if you call `.get()`.
3. Try a metadata `where` filter and observe the effect.
4. Clean up notebook 01: narrative markdown between cells, no dead code.

**Test.** **Restart kernel → Run All** on `Udaplay_01_solution_project.ipynb`. Zero errors, all
outputs present.

**Understand**
1. Why can a semantically perfect query still return a bad top result?
2. What distance range did *your* store produce for a hit vs a miss — and what threshold would you
   pick from that?
3. Why does `get()` have no distances?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P3 — `lib/` deep dive: messages, llm, tooling, state_machine · Day 2 · 2.0h · Status: `[ ]`

**Goal.** Understand the provided framework before building on it. **No deliverable code** — this
phase exists purely so P4–P6 aren't cargo-culting.

**Concepts**
- `@tool` reflection: type hints → JSON Schema, docstring → description. **The docstring is the
  prompt the model reads** — a wording bug is a behaviour bug.
- The `Step.run` schema filter and the `{**state}` spread — read `BUGS.md` **N1** and **N2** here.
  This is the single mechanic that explains the most confusing behaviour in the repo.
- The `Run` / `Snapshot` log: why deep-copying state per step is what makes P8's trajectory eval
  possible at all.
- Structured output via `client.beta.chat.completions.parse` and a Pydantic `response_format`.

**Build**
1. Write a throwaway `@tool` function; print `Tool.dict()` and read the generated schema.
2. Change only its docstring; re-print. Observe what the model would now see.
3. Build a 3-step toy `StateMachine`; inspect `run.snapshots`.
4. Deliberately return an undeclared key from a step. Watch it vanish. That's the filter.

**Test.** You can predict `Tool.dict()`'s output for a new function *before* running it, and you can
predict which keys survive a step return.

**Understand**
1. Trace a query through the agent's state machine, naming every step and both branch outcomes.
2. Why does an `Optional[str]` parameter become non-required in the schema?
3. `AgentState` has no `session_id` yet `state["session_id"]` works. Explain precisely why.

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P4 — The three tools · Day 3 · 2.5h · Status: `[ ]`

**Goal.** Implement `retrieve_game`, `evaluate_retrieval`, and `game_web_search` as `@tool`
functions integrated into the agent workflow. (Rubric: three tools, each a function/class.)

**Concepts**
- Tool descriptions are prompt engineering. The model chooses tools from docstrings alone.
- `evaluate_retrieval` is **LLM-as-judge**: it must return *structured* output (a Pydantic model
  with `useful: bool` + `description: str`) so the agent can branch on a field, not parse prose.
- Why the judge is a separate call rather than a threshold on cosine distance — distances aren't
  calibrated across query types.
- Tavily returns sources; those URLs are what satisfies the rubric's citation requirement, so
  capture them at the tool boundary.

**Build**
1. `retrieve_game(query: str)` → query the persistent store, return documents + metadata.
2. `evaluate_retrieval(question, retrieved_docs)` → LLM call with `response_format=`, returns the
   structured verdict.
3. `game_web_search(question)` → Tavily, returning answer **and** source URLs.
4. Test each in isolation before wiring any of them to an agent.

**Test.** Each tool called directly with a known input returns the expected shape.
`evaluate_retrieval` must return `useful=False` for the Mortal Kombat X retrieval — verify that
specifically, since the whole fallback depends on it.

**Understand**
1. What does the model literally receive when you pass it these three tools?
2. Why must `evaluate_retrieval` use structured output rather than returning a sentence?
3. What happens if retrieval is *good* but the judge says no — and how would you notice?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P5 — First working agent · Day 3 · 1.5h · Status: `[ ]`

**Goal.** An `Agent` that answers using internal knowledge first, evaluates the result, and falls
back to web search when needed.

**Concepts**
- System instructions as policy: the RAG → evaluate → fallback ordering is *prompted*, not
  hard-coded. Getting the model to reliably follow it is the actual work.
- The ReAct loop as executed by `tool_executor ⇄ llm_processor`.
- Why `temperature=0.0` for tool-calling reliability.

**Build**
1. Write the system instructions spelling out the tool-use policy.
2. Instantiate `Agent(model_name, instructions, tools)`.
3. Run all three canonical queries; print the full message trace.

**Test.** All three canonical smoke queries answer correctly. **Mortal Kombat X must visibly trip
the fallback** — retrieval miss, judge rejects, `game_web_search` fires, answer carries a citation.
Inspect `run.snapshots` to confirm the path rather than trusting the final text.

**Understand**
1. Which of the three queries used which tools — and how do you *prove* it from the run object?
2. What would make the agent skip `evaluate_retrieval` entirely, and how would you fix that?
3. How many LLM calls did the fallback query cost, and why that many?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P6 — Stateful agent + state machine · Day 4 · 2.0h · Status: `[ ]`

**Goal.** The agent maintains conversation state across multiple queries in a session. (Rubric:
*"can handle multiple queries in a session, remembering previous context"*; workflow implemented as
a state machine.)

**Concepts**
- `ShortTermMemory` as a session → `List[Run]` map; how `invoke` rehydrates `messages` from the
  previous run's final state ([agents.py:160-166](project/starter/lib/agents.py#L160-L166)).
- Session isolation: why two `session_id`s must not see each other's history.
- Context growth: every turn re-sends the full history. This is a cost and a limit, not a free
  feature.
- Revisit `BUGS.md` **N1** and **N2** now that you've seen state flow in practice.

**Build**
1. Ask a question, then a follow-up using a pronoun ("who published *it*?") in the same session.
2. Run the same follow-up in a *different* session; confirm it fails or asks for clarification.
3. Demonstrate `get_session_runs()` and `reset_session()`.
4. Decide on N1/N2 (add `session_id` to the schema, or leave it) and record the choice in §3.

**Test.** The pronoun follow-up resolves correctly in-session and does **not** resolve in a fresh
session. That contrast is the proof — a single successful follow-up proves nothing.

**Understand**
1. Where is conversation history physically stored between two `invoke` calls?
2. What is re-sent to the API on turn 3 of a session?
3. Why is `messages` in `AgentState` but `session_id` effectively isn't?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P7 — Long-term memory persistence · Day 4 · 1.5h · Status: `[ ]`

**Goal.** Persist facts learned from web searches so the agent "learns" across sessions.
(*Stand-out feature: Advanced Memory.*)

**Concepts**
- Short-term (in-process `Run` log, dies with the kernel) vs long-term (vector-backed, on disk).
- `MemoryFragment` and namespace/owner/timestamp filtering — Chroma `$and` where-clauses.
- Write-back: after a successful `game_web_search`, store the finding so the next run retrieves it
  from RAG instead of paying for another search.

**Build**
1. Hit and fix **B5** (`force=True` self-wipe) and **B6** (`get_namespaces` columnar iteration).
2. Instantiate `LongTermMemory` against the persistent manager.
3. Write findings back after a web-search hit.
4. Demonstrate retrieval of a stored fragment with a namespace filter.

**Test.** Ask a fallback question (Mortal Kombat X). Restart the kernel. Ask again — the answer now
comes from memory, with **no** Tavily call. Prove the absence of the call from the run trace, not by
timing.

**Understand**
1. Why did B5 fail *silently* while B1 and B4 raised — and which is more dangerous?
2. What would you store: the raw search result, or a distilled fact? Defend the choice.
3. How would stale memory hurt you here (e.g. "what is Rockstar working on right now")?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P8 — Evaluation · Day 5 · 1.5h · Status: `[ ]`

**Goal.** Measure prompt and tool-calling quality with `lib/evaluation.py::AgentEvaluator`.
(*Stand-out feature: Evals.*)

**Concepts**
- Three levels of evaluation: **final response** (was the answer right?), **single step** (was the
  right tool chosen?), **trajectory** (was the path efficient?). They catch different failures.
- LLM-as-judge and its failure modes — verbosity bias, self-preference, non-determinism.
- Why the trajectory eval needs the `Snapshot` log from P3.

**Build**
1. Assemble `TestCase`s, including at least one that *must* trigger the fallback.
2. Run `evaluate_final_response`, `evaluate_single_step`, `evaluate_trajectory`.
3. Tabulate the results.

**Test.** Evals run clean and produce a results table. Deliberately degrade a tool docstring, re-run,
and observe the score drop — that's what proves the eval measures anything at all.

**Understand**
1. Which failure would each of the three eval types catch that the others would miss?
2. What is your judge's own failure mode?
3. What did trajectory eval tell you about cost that the final-response score hid?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P9 — Stand-out features · Day 5 · 1.0h · Status: `[ ]`

**Goal.** Structured output alongside natural language, plus a personalised dataset.

**Concepts**
- Dual output: prose for humans, JSON for integration — one Pydantic model driving both.
- Citation provenance: distinguishing "from the local corpus" vs "from the web, with a URL".

**Build**
1. Define a `GameAnswer` Pydantic model (`answer`, `sources`, `confidence`, `used_web_search`).
2. Return it alongside the prose answer.
3. Add 3–5 games to `games/` and demonstrate a query only answerable from the additions.

**Test.** One query returns both a readable answer and valid JSON; a new-game query returns the
right answer from the corpus, not the web.

**Understand**
1. Why does structured output make the agent *composable*?
2. How is your `confidence` derived, and is it honest?

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

### P10 — Polish & submit · Day 5 · 1.5h · Status: `[ ]`

**Goal.** Submission-ready `project/`.

**Build**
1. Fill in `project/README.md` from the Udacity template — **in your own words**. Because
   `requirements.txt` lives at the repo root only, its Dependencies/Installation section must spell
   out the dependencies as text.
2. Confirm both solution notebooks: **Restart kernel → Run All**, zero errors, all outputs present.
3. Confirm the three example queries show reasoning, tool usage, and citations in the output.
4. Scrub any API key that leaked into a traceback or a printed payload.
5. Confirm `project/` contains only project files.
6. Complete §3 and §4 below.

**Test.** Clone the repo fresh into a temp directory, follow your own README from step 1, and reach
a working notebook. If you can't, the README is wrong.

**Gate.** Built `[ ]` ____ · Tested `[ ]` ____ · Explained `[ ]` ____

**Notes / blockers.**

---

## §3 Decision log

Every deviation from the starter code, dated, with rationale. Write it *as it happens* — this is
what you'll draw on when the panel asks "why did you do it that way?", and it never gets written
retroactively.

| Date | Phase | Decision | Rationale |
| --- | --- | --- | --- |
| | | | |

---

## §4 Submission checklist

- [ ] `project/starter/Udaplay_01_solution_project.ipynb` exists and is named exactly that
- [ ] `project/starter/Udaplay_02_solution_project.ipynb` exists and is named exactly that
- [ ] Both notebooks: restart-kernel → run-all, zero errors
- [ ] All cell outputs present and committed
- [ ] Persistent vector DB demonstrated (survives a kernel restart)
- [ ] Three tools implemented and integrated: `retrieve_game`, `evaluate_retrieval`, `game_web_search`
- [ ] Agent demonstrably tries RAG → evaluates → falls back to web search
- [ ] Multi-turn session context demonstrated
- [ ] ≥ 3 example queries with reasoning, tool usage, and final answer visible
- [ ] Citations present where a source exists
- [ ] `project/README.md` filled in, dependencies spelled out
- [ ] `project/` contains only project files
- [ ] No API keys anywhere in outputs, tracebacks, or committed files
- [ ] Every `lib/` fix logged in §3 and in the `BUGS.md` ledger
- [ ] You can explain every line without notes
