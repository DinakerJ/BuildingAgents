# UdaPlay — AI Game Research Agent

An AI research agent that answers natural-language questions about video games. It answers from a
local game corpus via RAG, judges whether that answer is actually good enough, and falls back to a
live web search when it isn't — then reports back with citations.

Built as a Udacity capstone. The interesting parts are the two-tier retrieval (RAG with an
LLM-as-judge gate in front of the web-search fallback), the agent's ReAct loop expressed as an
explicit state machine, and an evaluation harness that scores answers, tool choices, and whole
trajectories.

## Status

**Scaffolding only — implementation not started.**

Both notebooks are unmodified course starters: every implementation cell is a commented-out `# TODO`
and nothing has been executed. Real progress is tracked in [PROGRESS.md](PROGRESS.md).

## Repository layout

| Path | What it is |
| --- | --- |
| [project/](project/) | The submission artifact — notebooks, the 15-game corpus, and the provided `lib/` framework |
| [Agent.md](Agent.md) | Project charter: scenario, rubric, and the tutoring contract |
| [CLAUDE.md](CLAUDE.md) | Working notes: architecture, commands, conventions |
| [PROGRESS.md](PROGRESS.md) | 5-day sprint plan and phase gates |
| [BUGS.md](BUGS.md) | Known defects in the provided `lib/`, written up as coaching material |

## Which doc do I read?

| I want to know... | Read |
| --- | --- |
| What is being built and how it's graded | `Agent.md` |
| How `lib/` fits together; how to run things | `CLAUDE.md` |
| What's done, what's next, what's due today | `PROGRESS.md` |
| Why something in `lib/` is broken and how to fix it | `BUGS.md` |

## Quickstart

The checked-in `.venv` is Python 3.14 with nothing installed. Start over on 3.11 — ChromaDB 1.0.x
is unlikely to have 3.14 wheels.

```bash
uv venv --python 3.11
uv pip install -r requirements.txt
cd project/starter && jupyter lab
```

**The `cd` is not optional.** The notebooks use relative `lib.` imports and read the `"games"`
directory by relative path. Launched from anywhere else, they fail on the first import.

Create a `.env` in the repo root with these keys:

```
OPENAI_API_KEY=
OPENAI_BASE_URL=          # optional; only if routing through a gateway
CHROMA_OPENAI_API_KEY=    # used by ChromaDB's embedding function
TAVILY_API_KEY=
```

## Built with

Python 3.11 · ChromaDB · OpenAI (`gpt-4o-mini` + `text-embedding-3-small`) · Tavily · Pydantic ·
Jupyter

`.env` is gitignored. No credentials are stored in this repository.
