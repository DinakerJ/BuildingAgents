# UdaPlay — An AI Research Agent for the Video Game Industry

UdaPlay answers questions about video games. It first looks in a local vector database built
from a set of game records. If what it finds there is good enough, it answers from that. If not,
it searches the web and answers from those results instead, citing the URLs it used.

The point of the project is not the answers — it is the decision. The agent has to work out, on
its own, whether the local database actually answered the question or only returned something
that looked similar. Getting that judgement right is most of the work.

The project is delivered as two notebooks:

| Notebook | What it does |
| --- | --- |
| `starter/Udaplay_01_solution_project.ipynb` | Builds the vector database: client, embedding function, collection, ingest, and search |
| `starter/Udaplay_02_solution_project.ipynb` | Builds the agent: three tools, conversation state, long-term memory, evaluation, structured output |

---

## Getting Started

### Dependencies

There is no `requirements.txt` inside `project/` — the manifest lives at the repository root.
The packages needed are:

| Package | Minimum version | Why it is needed |
| --- | --- | --- |
| `chromadb` | 1.0.4 | The vector database. Stores the game documents and their embeddings on disk |
| `openai` | 1.73.0 | Calls `gpt-4o-mini` for the agent and `text-embedding-3-small` for embeddings |
| `pydantic` | 2.11.3 | Defines the structured output models and validates them |
| `python-dotenv` | 1.1.0 | Reads API keys from a `.env` file instead of hard-coding them |
| `tavily-python` | 0.5.4 | The web search fallback |
| `typing_extensions` | — | `lib/vector_db.py` and `lib/state_machine.py` import `TypedDict` from here |
| `pdfplumber` | — | Not used by UdaPlay, but `lib/vector_db.py` imports `lib/loaders.py`, which needs it |
| `ipykernel`, `jupyterlab` | — | To run the notebooks |

Python 3.11 is required. I originally had 3.14 and ChromaDB would not install — there are no
wheels for it at that version.

### Installation

```bash
# 1. Create the environment
uv venv --python 3.11

# 2. Install everything
uv pip install -r requirements.txt

# 3. Register the kernel so the notebooks can find it
uv run python -m ipykernel install --user --name udaplay --display-name "UdaPlay (3.11)"
```

Create a `.env` file at the repository root with these keys:

```
OPENAI_API_KEY=...
OPENAI_BASE_URL=...        # only if you are going through a gateway
TAVILY_API_KEY=...
```

`.env` is in `.gitignore` and no key is committed anywhere in this repository.

### Running the notebooks

Start Jupyter from inside `project/starter/`, not from the repository root:

```bash
cd project/starter
jupyter lab
```

This matters. The notebooks do `from lib.agents import Agent` and read the folder `games` by
relative path. Start Jupyter anywhere else and the imports fail. This was the single most common
thing that broke for me while building it.

Run notebook 01 first — it creates the database that notebook 02 reads.

---

## Testing

There is no pytest and no CI in this project. Testing means three things.

### 1. Restart the kernel and run every cell

This is the only check I trust, because it is exactly what a grader does. A notebook that works
when you run cells one at a time, in the order you happened to write them, is not finished. Both
notebooks pass a clean restart-and-run-all.

### 2. The three query cases

Each one proves something different, and the third is the one that matters most.

| Query | What it proves |
| --- | --- |
| "When was Pokémon Gold and Silver released?" | The straightforward case. The fact is in the database and retrieval finds it |
| "Which one was the first 3D platformer Mario game?" | The answer is not stated directly anywhere. The agent has to read across several descriptions and work it out |
| "Was Mortal Kombat X released for PlayStation 5?" | The fallback. This game is not in the database at all, so retrieval must come back with something wrong, the evaluation step must reject it, and only then should the web search fire |

The third query is the real test of the whole thing. The database will return *something* for it —
other fighting games, other PlayStation titles — with distances close enough to look reasonable.
If the agent answers from those, the evaluation step has failed and nobody gets an error message
about it. So I check the tool trace, not just the answer. If `game_web_search` does not appear in
the trace for that query, something is broken even if the answer happens to be right.

There is a second thing worth noticing in that answer. The web results for Mortal Kombat X are
full of pages saying "Mortal Kombat X PS5 gameplay", because the PS4 version runs on a PS5 through
backwards compatibility. Being released *for* a console and being playable *on* a console are two
different claims, and the web blurs them constantly. The agent's instructions tell it to keep
those apart, and it does — it says the game was released for PS4 in 2015 and is playable on PS5
through backwards compatibility. Getting that distinction right is the difference between a
correct answer and a confident wrong one.

### 3. The memory restart test

Ask the Mortal Kombat question once — the web search fires and the answer is written to disk.
Then restart the kernel, so nothing at all is left in memory: no agent, no session history, no
variables. Run everything again except the cell that clears memory, and ask the same question.

The second time, `game_web_search` does not fire. The answer comes back from what was stored on
disk. That is the proof that long-term memory works — there is no other channel between the two
kernels.

### 4. The evaluation harness

`lib/evaluation.py` gives three ways of scoring a run, and they catch different failures:

| Evaluator | Question it answers | What it looks at |
| --- | --- | --- |
| `evaluate_final_response` | Was the answer right? | The text output, scored by a second model |
| `evaluate_single_step` | Was the right tool picked? | One message's tool calls |
| `evaluate_trajectory` | Was the path sensible and what did it cost? | The whole recorded run — steps, tokens, tools |

All three test cases score 1.0 on the current build.

I also ran a deliberate degradation test: I replaced one tool's docstring with a vague one-liner
and re-ran the hardest query, expecting the score to drop. It did not — and that turned out to be
the more interesting result, for two reasons.

First, the agent still called the tool, because the system instructions name it explicitly by
step number. The docstring only matters when the instructions leave the ordering up to the model.

Second, the trajectory score stayed at 1.0 even though the tool that ran was not the one listed in
`expected_tools`, and the web search never fired at all. The check in `evaluate_trajectory` uses
`any()` — it passes if *at least one* expected tool appears. One correct tool call was enough to
mark the whole run successful. If I were extending this, changing that `any()` to `all()`, and
adding an ordering check, would be the first thing I did.

---

## Project Instructions

### Part 1 — The vector database

Notebook 01 builds a persistent ChromaDB collection called `udaplay` from the JSON files in
`games/`. Each game becomes one sentence:

```
[Platform] Name (Year) - Genre, published by Publisher. Description
```

I added genre and publisher to this sentence, which the starter template did not include. They
are useful search terms — a question like "which racing games do you have" has nothing to match
against without the genre in the text. The full record is also stored as metadata so it can be
filtered on exactly.

The ingest uses `upsert` rather than `add`. `add` fails if an ID already exists, so re-running the
cell crashes; `upsert` overwrites and the cell can be run as many times as you like.

Embeddings come from `text-embedding-3-small`, and the store is written to `project/starter/chromadb/`.

### Part 2 — The agent

**Three tools:**

- `retrieve_game` — semantic search over the collection, returns the three closest documents
- `evaluate_retrieval` — passes the question and those documents to a model and asks whether they
  actually answer it. Returns `useful: true/false` and a written reason
- `game_web_search` — Tavily search, used only when the evaluation says the documents are no good

**How the agent runs them in order.** This surprised me, so it is worth stating plainly: nothing
in the code enforces the order. `lib/agents.py` is a state machine, and the branch that decides
whether to loop only asks "did the model request any tool at all?" It cannot see which tools exist
or which one should come first. The ordering lives entirely in the system instructions, as numbered
steps.

I tested this by weakening the instructions to something vague, and the agent skipped
`evaluate_retrieval` completely and went straight to the web — with no error anywhere. So the
instructions are the only thing holding the procedure together, and rewording them casually is a
behaviour change, not an edit to a comment.

The same is true of tool docstrings. The `@tool` decorator reads a function's docstring and turns
it into the description the model sees when choosing tools. A docstring in this project is not a
comment for a human reader; it is part of the prompt.

**Conversation state.** Each call to the agent is recorded against a session. The next call in the
same session starts from the previous run's messages, which is what makes follow-up questions work.
I demonstrated this with the same follow-up question — "Who published it?" — asked twice: once in a
session that had history, where the agent correctly knew "it" meant Pokémon Gold and Silver, and
once in an empty session, where it had no idea what "it" referred to and listed publishers for
several unrelated games.

**Long-term memory.** Session history lives in a Python dictionary and dies with the kernel.
Long-term memory writes to a second ChromaDB collection, `long_term_memory`, in the same folder as
the game corpus. After any answer that required a web search, the agent's own answer is stored
there. `retrieve_game` then searches both the game corpus and this memory, marking remembered
results so they can be told apart.

I store the finished answer rather than the raw search results, because the answer is already the
useful part — the question and the fact, in a sentence. The raw results are mostly noise: ranking
scores, URLs, paragraphs about something else. Storing noise means retrieving noise later.

The obvious weakness is staleness. A fact that was true when it was stored stays in memory forever
and gets served back with the same confidence. Ask "what is Rockstar working on now" today and the
answer is wrong in six months, and nothing in the system knows that. `LongTermMemory.search()`
accepts a timestamp filter that would handle this; I have not wired it up, and I would want it
before this went anywhere real.

**Structured output.** Alongside the readable answer, the agent produces a `GameAnswer` object:

```json
{
  "answer": "...",
  "sources": ["https://..."],
  "confidence": "low",
  "used_web_search": true
}
```

This is done with a second model call that reads the prose answer and fills in the fields. I did it
this way deliberately. If I had forced structured output on the agent's own final call, the prose
would have been replaced by JSON and there would be nothing readable left. Two calls gives both:
the sentence for a person, the JSON for whatever consumes it next.

`confidence` is a plain string, `"high"` or `"low"`, not a number. The agent has no real probability
to report, and inventing one like `0.87` would suggest a precision that does not exist. "High" means
it came from the corpus we control; "low" means it came from the web and may be out of date.

**Extended dataset.** I added three games to the original fifteen — Zelda: Ocarina of Time, Red Dead
Redemption 2, and Halo: Combat Evolved — and confirmed the agent answers a question about Halo from
the corpus, with `used_web_search: false`. The ingest cell for these targets only the new files,
since re-embedding the original fifteen would cost API calls to produce identical vectors.

---

## Fixes made to the provided library

`lib/` was provided with the course. I changed four things in it, each one because the project could
not work otherwise. I have left the rest alone.

| Where | Problem | What I changed |
| --- | --- | --- |
| `lib/vector_db.py` | `chromadb.Client()` keeps everything in memory, so nothing survives a restart | Switched to `PersistentClient(path=...)` |
| `lib/vector_db.py` | The embedding function took Chroma's default model, `ada-002`, while the game corpus uses `text-embedding-3-small` | Passed `model_name` explicitly |
| `lib/memory.py` | `LongTermMemory.__init__` called `create_store(force=True)`, which deletes the collection. Building the object destroyed every stored memory before anything could read it | Default is now `get_or_create_store`; wiping requires `reset=True` |
| Notebook tool signature | A bare `list` type hint produces `"type": "string"` in the generated tool schema, so the model sent all the documents as one squashed blob | Used `list[str]` |

The two that cost me the most time were the ones that raised no error. A crash tells you where to
look. A silent failure lets everything keep running and look fine — the memory bug in particular
looked like it worked perfectly for a whole session, because writes succeeded; it was only the
reads, after a restart, that came back empty. I now assume that anything involving persistence is
lying to me until I have restarted the process and checked.

---

## Built With

* [ChromaDB](https://www.trychroma.com/) — vector database, persisted to disk
* [OpenAI](https://platform.openai.com/) — `gpt-4o-mini` for the agent and the judges,
  `text-embedding-3-small` for embeddings
* [Tavily](https://tavily.com/) — web search
* [Pydantic](https://docs.pydantic.dev/) — structured output and validation
* [Jupyter](https://jupyter.org/) — the notebooks themselves
* `lib/` — the small agent framework provided with the course: messages, tools, LLM wrapper,
  state machine, memory, and evaluation

## License

[License](../LICENSE.md)
