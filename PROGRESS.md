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
| 1 | P0 Environment & recon | 1.5 | 3.11 venv, deps installed, preflight cell green | `[x]` |
| 1 | P1 Persistent Chroma store + ingest | 2.0 | `udaplay` collection persisted on disk | `[x]` |
| 2 | P2 Semantic search → finish Part 1 | 1.5 | Notebook 01 restart-run-all clean | `[x]` |
| 2 | P3 `lib/` deep dive | 2.0 | You can explain how `@tool` becomes an OpenAI schema | `[x]` |
| 3 | P4 The three tools | 2.5 | `retrieve_game`, `evaluate_retrieval`, `game_web_search` | `[x]` |
| 3 | P5 First working agent | 1.5 | 3 smoke queries answered, incl. Tavily fallback | `[x]` |
| 4 | P6 Stateful agent + state machine | 2.0 | Multi-turn session demonstrably remembers context | `[x]` |
| 4 | P7 Long-term memory persistence | 1.5 | Web-search findings survive a kernel restart | `[ ]` |
| 5 | P8 Evaluation | 1.5 | AgentEvaluator: final-response, single-step, trajectory | `[ ]` |
| 5 | P9 Stand-out features | 1.0 | Structured JSON + citations; extended dataset | `[ ]` |
| 5 | P10 Polish & submit | 1.5 | `project/README.md` filled; both notebooks clean | `[ ]` |
| 5 | P9b *(optional, after P10)* | +0.6 | Tools moved to `lib/udaplay_tools.py`, notebook still green | `[ ]` |

Day totals: **D1** 3.5h · **D2** 3.5h · **D3** 4.0h · **D4** 3.5h · **D5** 4.0h

---

## §2 Phase blocks

### P0 — Environment & recon · Day 1 · 1.5h · Status: `[x]` GATED

**Goal.** A working Python 3.11 environment where every project dependency imports and both API
credentials are verified live.

**Concepts**
- Why 3.11 and not the 3.14 venv currently checked in (wheel availability for native deps).
- What `.env` + `python-dotenv` actually do, and how `OPENAI_BASE_URL` reaches *both* the chat client
  and Chroma's embedding client without being passed anywhere (both sit on the OpenAI SDK, which
  reads it from the environment). This is what disproved `BUGS.md` B3.
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
1. ~~Why does chat work through your gateway "for free" while embeddings might not?~~ *Premise was
   wrong — both work. Revised to: why do both inherit the gateway, and why test them separately
   anyway?* → Both clients sit on the OpenAI SDK, which reads `OPENAI_BASE_URL` itself. Tested
   separately because "shared library underneath" was an assumption, and assumptions are what the
   preflight exists to kill.
2. What exactly breaks if you launch Jupyter from the repo root instead of `project/starter/`?
   → `import lib` and `"games"` are relative, resolved against entry 1 of Python's search list,
   which is the folder Jupyter started in. `ModuleNotFoundError: No module named 'lib'`. Installed
   packages are unaffected — they sit at a fixed path inside `.venv`.
3. What is in `requirements.txt` that `Agent.md` didn't list, and why is it needed?
   → `typing_extensions` (`vector_db.py` imports `TypedDict` from it) and `pdfplumber` (pulled in
   via `vector_db.py` → `loaders.py`, even though UdaPlay never reads a PDF).

**Gate.** Built `[x]` 2026-09-09 · Tested `[x]` 2026-09-09 · Explained `[x]` 2026-09-09

**Notes / blockers.**
- Python 3.11.15. `openai` resolved to 3.10.0 and `chromadb` to 1.5.9, both well above the
  `Agent.md` pins. Checked the two APIs `lib/` depends on — `.beta.chat.completions.parse` and
  `PersistentClient` — both still present.
- Original `TAVILY_API_KEY` was invalid (15 chars, `Unauthorized`). Replaced; now passing.
- **B3 withdrawn.** The preflight disproved it — see `BUGS.md` B3.
- Preflight cell has no saved output yet; it was verified as a script. First real restart-run-all
  of notebook 01 happens at the end of P2.

---

### P1 — Persistent Chroma store + ingest 15 games · Day 1 · 2.0h · Status: `[x]` GATED

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
2. ~~Hit and fix B1, B2, B3 here.~~ **None of them surface in P1.** The notebook talks to ChromaDB
   directly, so it never touches `VectorStoreManager` where B1 and B2 live — those moved to **P7**,
   where `LongTermMemory` uses that class. B3 was withdrawn in P0.
3. Create the persistent client (`PersistentClient(path="chromadb")`).
4. Create the embedding function, passing `model_name` explicitly.
5. Create the collection with `get_or_create_collection`.
6. Build each game's sentence and load all 15 in one `upsert`.
7. Add a persistence-check cell using `get_collection` (raises if absent).

**Test.** Ingest, restart the kernel, then run **only** the imports cell and the persistence cell —
skipping the ingest. `count()` must still say 15. Pair it with `collection.count()` raising
`NameError` to prove memory really was wiped. One without the other proves nothing.

**Understand**
1. Why does `force=True` / delete-first exist at all, and when would you want it?
   → When the embedding model changes (old vectors sit in a different number-space and must all go),
   and to clear documents whose source files were deleted — `upsert` never removes anything.
2. If you changed the content template, what would you have to redo, and why?
   → Just edit and re-run, *because* we chose `upsert`. With the starter's `add()` the old sentences
   would silently survive and nothing would say so.
3. Where does the embedding actually get computed — your machine or the API?
   → OpenAI's servers. Text goes out, 1536 numbers come back. Hence the cost, the API key, and the
   need for a network. 1536 is fixed by `text-embedding-3-small`, not chosen by us.
4. Why `row["metadatas"][0]["Name"]` and not `row[0]["metadatas"]["Name"]`?
   → Chroma returns **columns, not rows** — a dict of parallel lists. Field name first, position
   second. `row[0]` is a `KeyError` because `row` is a dict with no key `0`. This is B6's mistake.

**Gate.** Built `[x]` 2026-09-14 · Tested `[x]` 2026-09-14 · Explained `[x]` 2026-09-14

**Notes / blockers.**
- Chose template **B** over the starter's: genre and publisher folded into the embedded sentence, so
  "which games are shooters" and "what did Nintendo publish" become answerable. Metadata still keeps
  the full JSON.
- Swapped `add()` for `upsert()`. Proved first that `add()` silently ignores an existing id and keeps
  the old text — a silent trap when the template changes.
- Batched all 15 into a single `upsert` rather than 15 separate calls.
- `text-embedding-3-small` passed explicitly. Chroma's default is the older `ada-002`, and both
  return 1536 numbers, so accepting the default would have been invisible.
- Observed distances on "first 3D Mario platformer": correct answer **0.4163**, wrong-but-related
  **0.6208**. Only 0.2 apart — direct evidence for why `evaluate_retrieval` needs a judge rather than
  a fixed distance threshold.

---

### P2 — Semantic search → finish Part 1 · Day 2 · 1.5h · Status: `[x]` GATED

**Goal.** Demonstrate semantic search over the collection and close out Part 1.

**Concepts**
- Cosine distance vs similarity, and why "distance ≈ 0.3" means nothing without calibration.
- Columnar results: `query()` returns `{ids, documents, metadatas, distances}` as parallel arrays
  wrapped one level deep (one per query text); `get()` returns them unwrapped and has **no**
  distances (`BUGS.md` B4).
- Why `n_results` is a recall/precision dial, and what it costs you downstream in prompt tokens.

**Build**
1. Query the store with the three canonical questions; print documents alongside distances.
2. ~~Hit and fix B4.~~ Not hit — the notebook calls ChromaDB's `get()` directly, which is correct.
   B4 is in `lib/`'s wrapper and still waits in P7.
3. Metadata `where` filters: exact match, numeric `$lt`, and filter combined with search.
4. Markdown headings between sections; title changed from `[STARTER]`.

**Test.** **Restart kernel → Run All**. Verified on disk: 12 code cells, execution counts 1–12 with
no gaps, zero errors, all outputs present.

**Understand**
1. Why can a semantically perfect query still return a bad top result?
   → The database has no concept of good or bad. It ranks by distance and returns the nearest,
   whether or not anything actually answers the question.
2. What distance range for a hit vs a miss, and what threshold would you pick?
   → **No threshold works.** "violent fighting game with fatalities" → Smash Bros at **0.6168** is a
   *useful* result; "Mortal Kombat X on PS5" → Spider-Man 2 at **0.5177** is *useless*. The useless
   one scores better. Any cutoff either rejects the good answer or accepts the bad one. Distance
   measures *how related*; the question is *does this answer it*. Different quantities.
3. Why does `get()` have no distances?
   → A distance needs two points. `query()` has your question as the second point; `get()` has no
   question at all, so there is nothing to measure from. This is exactly what B4 gets wrong.
4. Why were the filtered and unfiltered searches both 0.4939 for Gran Turismo?
   → Distance is computed between the question and one document, and depends on nothing else. The
   filter only decides which documents are eligible to be returned. Removing 13 games does not move
   the remaining one. This is what makes scores comparable across different searches.

**Gate.** Built `[x]` 2026-09-15 · Tested `[x]` 2026-09-15 · Explained `[x]` 2026-09-15

**Notes / blockers.**
- **Part 1 of the rubric is complete**: JSON loaded and formatted, persistent store with embeddings,
  semantic search demonstrated.
- Key measurements, all reproducible in the notebook output:

  | Query | Top result | Distance | Actually useful? |
  | --- | --- | --- | --- |
  | Pokémon Gold and Silver release date | Pokémon Gold and Silver | 0.3424 | yes |
  | first 3D platformer Mario game | Super Mario 64 | 0.4062 | yes |
  | *(same, 2nd result)* | Super Mario World | 0.4338 | no — 0.028 away |
  | Mortal Kombat X on PS5 | Marvel's Spider-Man 2 | 0.5177 | **no — game absent** |
  | violent fighting game with fatalities | Super Smash Bros. Melee | 0.6168 | **yes** |

- Found a data quirk worth keeping: publishers are stored as **7 distinct strings** including
  `Sony Computer Entertainment` *and* `Sony Interactive Entertainment`, plus
  `Microsoft Game Studios` *and* `Xbox Game Studios`. An exact filter on one name finds half the
  Sony games. Decided **not** to normalise — the agent works in natural language and would never
  produce either exact string, so the fix would serve a user who does not exist. Recorded as a
  "how would you fix this" answer rather than a change.

---

### P3 — `lib/` deep dive: messages, llm, tooling, state_machine · Day 2 · 2.0h · Status: `[x]` GATED

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

**Build** (all in scratchpad — nothing written into `project/`)
1. `@tool` on a throwaway function; printed `Tool.dict()` and mapped each JSON field back to its
   Python source.
2. A 3-step toy `StateMachine` (`ShopState` with `basket` / `total`); printed every snapshot.
3. Returned an undeclared key `discount` from a step and watched it disappear.
4. Structured output: the same judging prompt with and without `response_format`, using the real
   Mortal Kombat X / Spider-Man case from P2.

**Test.** Can predict which keys survive a step return, and which parts of a function end up in the
tool schema.

**Understand**
1. What does the model receive when you give it a tool?
   → Four fields from `Tool.dict()`: `name` (function name), `description` (**the docstring**),
   `properties` (parameter names + type hints), `required` (params with no default). **Not** `func`
   — the function itself is stored on the object but never sent.
2. A step returns a key not in the schema; a schema key is not returned. What happens to each?
   → Returned but not in schema → **silently dropped**. In schema but not returned → **kept
   unchanged** (not emptied). The schema governs what a step may *change*, not what the state may
   *contain*.
3. Why must the judge use `response_format` instead of returning a sentence?
   → The agent has to branch on a real boolean. Prose would mean string-matching for "No", which
   breaks on rephrasing and makes the fallback fire at random.
4. What type is `r.content` after a `response_format` call?
   → A **string** of JSON. `lib/llm.py:76-80` keeps only `message.content` and discards OpenAI's
   ready-made `message.parsed`. Need `Model.model_validate_json(r.content)`, or
   `PydanticOutputParser` from `lib/parsers.py:34-38`. Skipping it does not crash —
   `bool('{"useful":false}')` is `True`, so the fallback would silently never fire.

**Gate.** Built `[x]` 2026-09-16 · Tested `[x]` 2026-09-16 · Explained `[x]` 2026-09-16

**Notes / blockers.**
- **N1 deferred to P6.** The underlying mechanic (schema filter + `{**state}` carry-forward) is
  understood and demonstrated. Applying it to `AgentState`/`session_id` needs a running agent to be
  meaningful, which does not exist until P6.
- Corrected a wrong claim in `CLAUDE.md`: `lib/parsers.py` is **not** part of the PDF path. It holds
  output parsers, and `PydanticOutputParser` is the string→object step needed after every
  `response_format` call. Already used at `lib/evaluation.py:102`.
- `evaluate_retrieval` **does not exist yet** — it is a `# TODO` at notebook 02 line 158, written in
  P4. The working `response_format` example to pattern-match is
  `lib/evaluation.py:96-104`.

---

### P4 — The three tools · Day 3 · 2.5h · Status: `[x]`

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

**Gate.** Built `[x]` 2026-09-17 · Tested `[x]` 2026-09-17 · Explained `[x]` 2026-09-17

**Notes / blockers.**
- Starter docstring for `game_web_search` was a copy-paste of `retrieve_game`'s ("Finds most
  results in the vector DB"). Rewritten to describe web search and to name itself the fallback.
  The docstring is the tool description the model sees, so this was a behaviour fix.
- `EvaluationReport` did not exist in `lib/` — defined in the notebook, patterned on
  `JudgeEvaluation` at `lib/evaluation.py:57`.
- Smoke tests call `.func(...)` to bypass the `Tool` wrapper and invoke the plain function.
- Verified: `useful=True` for the Pokémon retrieval, `useful=False` for Mortal Kombat X.
- Web results for MK X include a "PS5 Gameplay" video title and a PS Store backwards-compatibility
  notice. Correct answer is PS4/Xbox One/PC, 2015 — playable on PS5, not released for it. The
  wording risk moves to the agent's system prompt in P5.

---

### P5 — First working agent · Day 3 · 1.5h · Status: `[x]`

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

**Gate.** Built `[x]` 2026-09-17 · Tested `[x]` 2026-09-17 · Explained `[x]` 2026-09-17

**Results.** Each query in its own session, so none could answer from another's history.

| Query | Tools fired | Snapshots | Tokens (agent only) |
| --- | --- | --- | --- |
| Pokémon Gold and Silver | retrieve → evaluate | 7 | 2,235 |
| First 3D Mario platformer | retrieve → evaluate | 7 | 2,456 |
| Mortal Kombat X on PS5 | retrieve → evaluate → **web** | 9 | 6,735 |

- The fallback is proven from the tool trace, not from the answer text. `tools_used()` reads
  `run.get_final_state()["messages"]`, which the machine wrote at `state_machine.py:239` — the
  answer text is the model's claim, the snapshots are the machine's record.
- Q3's answer held the backwards-compatibility distinction ("not released for PS5, playable
  through backwards compatibility") against web results including a video titled "MK X - PS5
  Gameplay". That line in the instructions earned its place.
- Q2 is RAG + reasoning: the corpus never contains the word "first". The agent derived it from
  "groundbreaking 3D platformer" plus 1996 being earliest, and the judge accepted the documents.
- Snapshot count excludes termination — `state_machine.py:226-228` breaks before saving.
- **Q3 cost five model calls, not four.** Four `llm_processor` turns plus the judge's own call
  inside `evaluate_retrieval`. `total_tokens` only sums the agent's calls
  (`agents.py:70-72`), so the reported figure undercounts — a tool that calls a model spends
  tokens the agent never sees.

**Demonstrated, not assumed: instructions are persuasion, not enforcement.** Ran the same agent,
same tools, same model, `temperature=0.0`, changing only the instructions:

| Instructions | Pokémon | Mortal Kombat X |
| --- | --- | --- |
| Strict (numbered procedure) | retrieve → evaluate | retrieve → evaluate → web |
| Vague ("use them as needed") | retrieve → **web** | retrieve → **web** |

The vague run skipped the judge on both queries and went to the web for a question the local
corpus answers. No error either time. `check_tool_calls` at `agents.py:133-137` only asks whether
the model requested *any* tool — it cannot see which tools exist or verify an order. Guaranteeing
the sequence means wiring it as named steps, which is the optional task in cell `eb83fbb1`.

**Notes / blockers.**
- Notebook execution counters are out of order (1–11, then 14). Needs a restart-run-all before
  submission; tracked in §4.

---

### P6 — Stateful agent + state machine · Day 4 · 2.0h · Status: `[x]`

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

**Gate.** Built `[x]` 2026-09-18 · Tested `[x]` 2026-09-18 · Explained `[x]` 2026-09-18

**Results.** The proof is the contrast, not the successful follow-up. Same three words, two sessions:

| Session | Question | Came in with | Answer |
| --- | --- | --- | --- |
| `chat` turn 1 | "When was Pokémon Gold and Silver released?" | 0 messages | 1999, Game Boy Color |
| `chat` turn 2 | "Who published it?" | **7 messages** | "published by Nintendo" |
| `cold` turn 1 | "Who published it?" | 0 messages | lists three unrelated games, asks which one |

- **The evidence is the tool call, not the answer.** On turn 2 the model's `retrieve_game` query
  came back with Pokémon — but the word "Pokémon" was never in that turn's question. It resolved
  "it" from the carried-in messages *before* searching. Printing the 13-message pile with the
  turn boundary marked is what made this visible.
- The cold session degraded well rather than guessing, which is the `Do not guess` instruction
  earning its place.
- Cost: a four-word follow-up cost ~52% more than the full question before it (2,201 → 3,352),
  because the whole pile is re-sent every turn.
- N1 decision recorded in §3: left alone.

**The one idea this phase turns on.** `state_machine.py:49-53` does two separate things —
`updated = {**state}` copies everything forward, then the schema filter decides which *returned*
keys are accepted. So the schema governs what a step may **change**, not what the state may
**hold**. `messages` is in the schema because every step appends to it; `session_id` is not
because nothing ever changes it.

**Notes / blockers.**
- Session isolation happens in `invoke` before the machine starts — a dictionary lookup by key at
  `agents.py:158-162`. It has nothing to do with `AgentState`.
- Nothing here survives a kernel restart: `ShortTermMemory.sessions` is a plain dict in process
  memory. That gap is what P7 addresses.

---

### P7 — Long-term memory persistence · Day 4 · 1.5h · Status: `[x] GATED 2026-09-22`

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

**Gate.** Built `[x]` 2026-09-22 · Tested `[x]` 2026-09-22 · Explained `[x]` 2026-09-22

**Notes / blockers.**
- Fixed B2 (ephemeral client → PersistentClient), B5 (force-wipe self-bug → opt-in reset), B8 (wrong embedding model → text-embedding-3-small).
- Write-back stores the agent's distilled answer (not raw Tavily results). TimestampFilter available but not wired — deferred.
- Restart test confirmed: 1 fact survived kernel restart; game_web_search absent from second run.

---

### P8 — Evaluation · Day 5 · 1.5h · Status: `[x] GATED 2026-09-22`

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

**Gate.** Built `[x]` 2026-09-22 · Tested `[x]` 2026-09-22 · Explained `[x]` 2026-09-22

**Notes / blockers.**
- Three TestCases covering corpus hit, reasoning, and fallback path.
- All three traj scores 1.0 on baseline run.
- Degradation test: vague docstring did not change tool path (instructions named tool explicitly); traj score stayed 1.0 due to `any()` check in evaluate_trajectory — evaluator's own blind spot identified.
- `evaluate_single_step` limitation noted: checks last AI message only, misses multi-step ordering.

---

### P9 — Stand-out features · Day 5 · 1.0h · Status: `[x] GATED 2026-09-23`

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

**Gate.** Built `[x]` 2026-09-23 · Tested `[x]` 2026-09-23 · Explained `[x]` 2026-09-23

**Notes / blockers.**
- `GameAnswer` model: answer, sources, confidence, used_web_search. Second LLM call extracts fields from prose — keeps prose intact for humans, JSON available for downstream systems.
- Extended dataset: games/016 (Zelda OoT), 017 (RDR2), 018 (Halo CE). Ingest targets only new files with upsert; collection now 18 documents.
- Halo query answered with used_web_search=false — proves new documents embedded and retrieved from corpus.

---

#### P9b — OPTIONAL: move the three tools into a `.py` module · +30–40 min · Status: `[ ]`

**Rules of engagement.** Not required by the rubric. **Do P10 first** and only start this with both
notebooks already passing restart-run-all. If it breaks, revert and lose nothing. Commit before
starting so revert is one command.

**Goal.** Move `retrieve_game`, `evaluate_retrieval`, and `game_web_search` out of notebook 02 into
`lib/udaplay_tools.py`, leaving the notebook as: import, run three queries, show results.

**Concepts**
- Python's import search list: entry 1 is the folder you started Jupyter in for a notebook, but the
  folder the *file lives in* when you run `python file.py`. Same list, different first entry.
- A `.py` file can't reach notebook variables. Anything it needs — the collection, the client — must
  be passed in as an argument or built inside the module.
- `if __name__ == "__main__"`: code that runs when the file is executed directly but not when it's
  imported. Lets the module self-test.
- Why the notebook gets *shorter and more readable*, which is what a grader sees first.

**Build**
1. Create `lib/udaplay_tools.py`; move the three `@tool` functions across with their docstrings
   intact — the docstrings are the model's tool descriptions, so a typo here is a behaviour change.
2. Replace the notebook's hidden dependency on a global `collection` with an explicit parameter or a
   module-level factory function.
3. In notebook 02, swap the tool definitions for `from lib.udaplay_tools import ...`.
4. Add an `if __name__ == "__main__"` block that calls each tool once, so
   `python lib/udaplay_tools.py` smoke-tests the module on its own.

**Test.** Restart kernel → run all on notebook 02. Same answers, same citations as before the move.
Then run `python lib/udaplay_tools.py` from `project/starter/` and confirm it works standalone.

**Understand**
1. Why did `import lib` fail from the repo root but `import chromadb` succeed?
2. What broke (or nearly broke) when the tools stopped sharing the notebook's variables?
3. What does `if __name__ == "__main__"` actually guard against?

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
| 2026-09-09 | P0 | Rebuilt venv on Python 3.11.15, discarding the 3.14 one | `chromadb` ships compiled code and needs a matching prebuilt wheel; 3.14 had none, so pip would try to build from source and fail |
| 2026-09-09 | P0 | Accepted `openai` 3.10.0 and `chromadb` 1.5.9 rather than pinning to the `Agent.md` minimums | Verified the only two APIs `lib/` relies on still exist: `.beta.chat.completions.parse` and `PersistentClient`. Pinning down to 1.x would have been busywork |
| 2026-09-09 | P0 | Added `typing_extensions` and `pdfplumber` to `requirements.txt` | Both are imported by `lib/` but missing from the `Agent.md` list. `pdfplumber` arrives via `vector_db.py` → `loaders.py`, so it's needed even though no PDF is ever read |
| 2026-09-09 | P0 | **Withdrew B3** instead of applying the documented fix | Preflight showed Chroma defaults `api_key_env_var` to `OPENAI_API_KEY` and inherits `OPENAI_BASE_URL` through the OpenAI SDK. Predicted bug, disproved by evidence. Adding `api_base` would have been an unjustifiable change |
| 2026-09-09 | P0 | Replaced the Tavily API key | Original was 15 chars and returned `Unauthorized`; looked truncated |
| 2026-09-09 | P0 | Wrote the preflight to report all checks independently rather than assert-and-stop | One run surfaced the broken Tavily key *and* confirmed the other four services, which is what proved P1/P2 weren't blocked |
| 2026-09-09 | P9b | Added an optional notebook → `.py` refactor after P10 | Wanted the `.py` transition as a learning goal. Sequenced after submission polish so a late break costs nothing |
| 2026-09-14 | P1 | Extended the starter's content template to include `Genre` and `Publisher` | Metadata is never embedded, so anything absent from the sentence cannot be found by a meaning search. Checked the data first: 4 of 15 descriptions never mention the game's own name and 12 of 15 never mention the platform |
| 2026-09-14 | P1 | Used `upsert()` where the starter used `add()` | Demonstrated that `add()` silently keeps the old text when an id already exists. Editing the template and re-running would have left stale sentences in the store with no warning |
| 2026-09-14 | P1 | Batched all 15 documents into one `upsert` instead of one call per file | The embedding function accepts a list, so this is a single API round trip rather than fifteen |
| 2026-09-14 | P1 | Passed `model_name="text-embedding-3-small"` explicitly | Chroma defaults to `text-embedding-ada-002`. Both return 1536 numbers, so taking the default would have silently given worse search quality with nothing to notice |
| 2026-09-14 | P1 | Used `get_or_create_collection`, and `get_collection` for the persistence check | `get_or_create` makes the cell safe to re-run. `get_collection` raises when absent, which is what makes the persistence check meaningful rather than returning an empty collection |
| 2026-09-14 | P1 | Moved B1 and B2 from P1 to P7 | The notebook uses ChromaDB directly and never constructs `VectorStoreManager`, so neither bug can surface until `LongTermMemory` needs that class |
| 2026-09-15 | P2 | Rejected a distance threshold for retrieval quality, on measured evidence | A useful result scored 0.6168 while a useless one scored 0.5177. No cutoff separates them, so `evaluate_retrieval` has to read the text rather than compare a number |
| 2026-09-15 | P2 | Left the duplicate Sony / Microsoft publisher names unnormalised | Adding a derived `PublisherGroup` field would work, but the agent takes natural-language questions and would never emit either exact string. Filtering on publisher is not on its path |
| 2026-09-17 | P4 | Rewrote the `game_web_search` docstring; the starter's was a copy of `retrieve_game`'s | `tooling.py:24` makes the docstring the tool description the model sees. Two tools both claiming to search the vector DB gives the model no basis to pick the fallback. A wording change here is a behaviour change |
| 2026-09-17 | P4 | Defined `EvaluationReport` in the notebook rather than in `lib/` | The starter TODO names the model but it exists nowhere in `lib/`. Kept it beside the tool that uses it; patterned on `JudgeEvaluation` at `lib/evaluation.py:57` |
| 2026-09-17 | P4 | `evaluate_retrieval` returns a plain `dict`, not the `EvaluationReport` object | The tool result is serialised into a `ToolMessage`. A dict serialises as-is; a Pydantic object would need an extra `.model_dump()` on the way out |
| 2026-09-17 | P4 | `retrieve_game` returns documents only, no distances | Distances were shown in P2 to be uncorrelated with usefulness, and the judge reads text. Passing a number the judge cannot act on would only invite a threshold |
| 2026-09-17 | P4 | Tested each tool through `.func(...)` rather than the `Tool` wrapper | `@tool` returns a `Tool` object whose call path expects JSON arguments from the model. `.func` is the original function, which is what an isolation test should exercise |
| 2026-09-17 | P5 | Tightened `retrieved_docs: list` to `list[str]` (B7) | `tooling.py:61` tests `get_origin(typ) is list`, which is `None` for a bare `list`, so the schema silently defaulted to `"string"`. The model was being asked to flatten three documents into one blob of text itself. Found by printing `evaluate_retrieval.dict()`; verified by diffing the schema before and after |
| 2026-09-17 | P5 | Set `temperature=0.0`, overriding the `Agent` default of `0.7` | Tool selection is a choice the model makes, so randomness there makes the tool-use policy untestable. `agents.py:23` |
| 2026-09-17 | P5 | Put the RAG → evaluate → fallback ordering in the system instructions, not in code | Nothing in `agents.py` knows one tool from another; `_tool_step` matches by name and the only branch asks whether any tool was requested. Prose is the only place the policy can live, and it is restated as a prohibition because the docstrings say the same thing from the tool side |
| 2026-09-18 | P6 | **Left N1 alone** — did not add `session_id` to the `AgentState` schema | Session isolation happens in `invoke` before the machine starts (`agents.py:158-162`, a dictionary lookup by key) and does not involve the schema at all. Adding the key would only make each step's `session_id` return survive the filter instead of being dropped — and since every step returns the value it was handed, nothing observable changes. Seeding it in `initial_state` *is* load-bearing, because `agents.py:55` reads it with square brackets |
| 2026-09-18 | P7 | **Fixed B2** — `VectorStoreManager.__init__` now builds `chromadb.PersistentClient(path=persist_path)`, default `"chromadb"` | `chromadb.Client()` keeps everything in process memory, so "long-term" memory died with the kernel while looking like it worked all session. `LongTermMemory` has no way to reach disk on its own — `memory.py:225` requires a `VectorStoreManager`, so the store it gets is whatever that class built. Added `persist_path` as a defaulted argument so existing calls are unchanged. Verified by writing in one process and reading in a second |
| 2026-09-18 | P7 | **Fixed B5** — `LongTermMemory.__init__` now uses `get_or_create_store`, with the destructive path behind `reset=False` | `create_store(force=True)` deletes the collection first, so constructing the object destroyed every stored memory before anything could read it. It failed silently *and* destroyed data, and because the wipe is in `__init__` the act of inspecting the store destroyed the evidence — no amount of print-debugging inside `search()` could have found it. Verified: fact stored, object rebuilt twice, count stayed 1; `reset=True` still returns 0, which is exactly what every construction used to do |
| 2026-09-18 | P7 | **Fixed B8** — passed `model_name="text-embedding-3-small"` in `_create_embedding_function`, and dropped/rebuilt `long_term_memory` | The manager took Chroma's `ada-002` default while the corpus uses `3-small`, so the two collections in one folder were embedded by different models. Both return 1536 numbers and each collection is self-consistent, so nothing failed — memory searches were just quietly worse. Rebuild was required because mixing two embedding models in one collection makes distances meaningless |
| 2026-09-18 | P6 | Gave `tools_used()` a `this_turn_only=True` default, slicing off the carried-in messages | Walking the whole message list counts the previous turn's tools as the current turn's. Harmless in P5 (one session per query, nothing carried in) but wrong from the second turn onward. `lib/evaluation.py:262-265` has the same walk-all pattern, so P8 trajectory scoring will over-report tools on multi-turn sessions unless it is sliced the same way |

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
