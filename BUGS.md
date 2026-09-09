# BUGS.md — Known defects in the provided `lib/`

The `project/starter/lib/` framework was shipped by the course, not written by you. It has real
defects. Finding, explaining, and fixing them is **panel ammunition** — "I debugged the framework I
was given" is a stronger story than "I filled in the TODOs."

**These fixes are documented, not pre-applied.** Each one gets applied live during the phase that
hits it, so you build it yourself and can explain it. Log each application in `PROGRESS.md` §3.

Every entry follows the same shape:

> **Symptom** (what you see) → **Root cause** (`file:line`) → **Why it happens** (the underlying
> mechanic) → **Impact** (what it costs you if unfixed) → **The fix** → **Panel answer**

**When the sprint reaches a bug, it gets called out before it gets fixed** — name it, explain why
it's wrong, state the blast radius, *then* patch. Never a silent fix. The `Impact` line is the one
that matters most: two of these fail loudly and one fails silently, and the silent one is the
dangerous one.

Ordered by when the sprint will hit them.

---

## B1 — `create_store` raises `UnboundLocalError` and hides the real error

**Hit in:** P1 · **Severity:** high (masks every other store error)

**Symptom.** You call `manager.create_store("udaplay")` a second time. Instead of a useful
"collection already exists" message you get a printed hint followed by
`UnboundLocalError: cannot access local variable 'chroma_collection'`.

**Root cause.** [vector_db.py:181-189](project/starter/lib/vector_db.py#L181-L189):

```python
try:
    chroma_collection = self.chroma_client.create_collection(...)
except Exception as e:
    print(f"Pass `force=True` or use `get_or_create_store` method")

return VectorStore(chroma_collection)   # <-- runs even when the try failed
```

**Why it happens.** The `except` swallows the exception but does not `return` or `raise`. Control
falls through to the `return`, where `chroma_collection` was never bound because the assignment
inside `try` never completed. `e` is captured and never used, so the actual ChromaDB error — the one
that tells you *what* went wrong — is discarded.

This is the classic "log and continue" antipattern: the handler converts a precise, actionable
exception into a vague one thrown from a different line.

**Impact.** Fails **loudly**, so you can't ship past it — but it costs you *debugging time*, not
correctness. The damage is diagnostic: every future store error in P1, P2, and P7 arrives disguised
as an `UnboundLocalError` about a variable that has nothing to do with the real problem. Left
unfixed, you will lose 20 minutes to a wrong hypothesis at least once. It's first in the list
because fixing it makes B2 and B3 legible.

**The fix.** Re-raise with context instead of falling through:

```python
except Exception as e:
    raise RuntimeError(
        f"Could not create store '{store_name}': {e}. "
        f"Pass force=True or use get_or_create_store()."
    ) from e
```

**Panel answer.** "The handler logged and continued, so a `create_collection` failure surfaced two
lines later as an `UnboundLocalError` about an unrelated variable. I re-raised with `from e` so the
original ChromaDB error stays in the traceback chain."

---

## B2 — `VectorStoreManager` uses an ephemeral client; the rubric requires persistence

**Hit in:** P1 · **Severity:** high (rubric requirement)

**Symptom.** You ingest all 15 games, restart the kernel, and the collection is empty. Nothing was
ever written to disk.

**Root cause.** [vector_db.py:158](project/starter/lib/vector_db.py#L158):

```python
self.chroma_client = chromadb.Client()
```

**Why it happens.** `chromadb.Client()` is the **in-memory** client. It lives and dies with the
Python process. `chromadb.PersistentClient(path=...)` is the one that writes a SQLite file plus the
index to disk. The rubric says *"The processed data is added to a **persistent** vector database"*,
and notebook 01 hints at `PersistentClient(path="chromadb")`.

**Impact.** Fails **silently, and it's a graded failure.** The notebook looks perfect — ingest runs,
count is 15, queries work — right up until the kernel restarts. Two consequences: (1) the rubric's
persistence requirement is unmet, which is a direct score loss; (2) every downstream phase pays to
re-embed all 15 games on every kernel restart, which is real money and real waiting. It also
pre-poisons P7: long-term memory built on an ephemeral client can never persist anything, so you'd
be debugging B5 while B2 is the actual cause.

**The fix.** Give the manager an optional path and branch:

```python
def __init__(self, openai_api_key: str, persist_path: Optional[str] = None):
    self.chroma_client = (
        chromadb.PersistentClient(path=persist_path) if persist_path
        else chromadb.Client()
    )
```

Keeping the ephemeral branch matters — tests and throwaway experiments should not litter the disk.

**Panel answer.** "The provided manager hard-coded the in-memory client, which can't satisfy a
persistence requirement. I made the path optional so the same class covers both the graded
persistent store and disposable in-memory use."

---

## B3 — ~~Embeddings ignore `OPENAI_BASE_URL`~~ — WITHDRAWN, does not apply

**Status:** **Not a bug on chromadb 1.5.9.** Predicted from reading the code, then disproved by the
P0 preflight on 2026-09-09. Kept here because the *reasoning* is still worth knowing and because
withdrawing a wrong diagnosis on evidence is a better panel story than quietly deleting it.

**What was predicted.** That [vector_db.py:161-165](project/starter/lib/vector_db.py#L161-L165)
passing only `api_key` — no `api_base` — would send embeddings to `api.openai.com` while chat went
through the `OPENAI_BASE_URL` gateway. And that the missing `CHROMA_OPENAI_API_KEY` from
`project/starter/README.md` would compound it.

**What the preflight actually found.** Both wrong, for two separate reasons:

| Prediction | Reality on chromadb 1.5.9 |
| --- | --- |
| Needs `CHROMA_OPENAI_API_KEY` | `api_key_env_var` **defaults to `OPENAI_API_KEY`**. The bare `OpenAIEmbeddingFunction()` constructor works with no arguments at all. |
| Needs explicit `api_base` | `api_base` is `None` in the config and embeddings still return 1536 dims through `https://openai.vocareum.com/v1`. Chroma builds its client on the OpenAI SDK, which reads `OPENAI_BASE_URL` from the environment itself. |

So both clients inherit the gateway from the same env var. There is no split.

**Why the prediction was reasonable anyway.** Older chromadb releases *did* default to
`CHROMA_OPENAI_API_KEY` — which is exactly why `project/starter/README.md` still asks for it. The
starter README is stale, not wrong-at-the-time. The general principle also holds: two separate
client objects can absolutely diverge on configuration, and "chat works" genuinely does not prove
"embeddings work". That is why the preflight tests them as two independent checks.

**Impact.** None. No fix needed. Do not add `api_base` — it is already handled, and hardcoding it
would break the moment the gateway changes.

**Action.** `CHROMA_OPENAI_API_KEY` can stay absent from `.env`. Ignore that line in the starter
README.

**Panel answer.** "I predicted a config split between the chat client and the embedding client,
because they're separate objects and only one obviously read the gateway env var. I wrote the
preflight to test them independently, and it disproved me — this chromadb version builds its
embedding client on the OpenAI SDK, so both inherit `OPENAI_BASE_URL`. I withdrew the fix rather
than apply a change I couldn't justify."

---

## B4 — `VectorStore.get()` passes an `include` value that `get()` doesn't accept

**Hit in:** P2 · **Severity:** medium (every `.get()` call raises)

**Symptom.** Any call to `store.get(...)` raises a ChromaDB validation error about `distances`.

**Root cause.** [vector_db.py:134-139](project/starter/lib/vector_db.py#L134-L139):

```python
return self._collection.get(
    ids=ids, where=where, limit=limit,
    include=['documents', 'distances', 'metadatas']
)
```

**Why it happens.** `distances` only exists as a result of a **similarity search**. `query()`
embeds your text and computes a distance per result, so `distances` is a valid include there
([vector_db.py:105](project/starter/lib/vector_db.py#L105) — correct). `get()` is a direct lookup by
id or metadata filter; no query vector exists, so there is nothing to measure distance *from*.

Note the copy-paste lineage: `query()` above it has the identical `include` list, and it's right
there. Someone duplicated the line into a method where one element became meaningless.

**Impact.** Fails **loudly**, and the blast radius is narrower than it looks: `query()` is
unaffected, so semantic search — the thing Part 1 is graded on — works fine without this fix. You
only hit it if you call `.get()`. But B6 calls it, so this silently becomes a P7 blocker too: fix
B4 and B6 still fails, which makes it easy to conclude the fix "didn't work". Fix them together.

**The fix.** Drop `distances`:

```python
include=['documents', 'metadatas']
```

**Panel answer.** "`distances` is only defined relative to a query vector. `get()` is a keyed
lookup with no query vector, so Chroma rejects the include. It was copy-pasted from `query()` five
lines above."

---

## B5 — `LongTermMemory` wipes itself on every construction

**Hit in:** P7 · **Severity:** high (silently defeats the whole feature)

**Symptom.** The agent learns a fact from a web search. You restart the kernel, rebuild the agent,
ask again — and it's gone. No error, no warning. Long-term memory that never remembers anything.

**Root cause.** [memory.py:226](project/starter/lib/memory.py#L226):

```python
def __init__(self, db: VectorStoreManager):
    self.vector_store = db.create_store("long_term_memory", force=True)
```

**Why it happens.** `force=True` routes through `delete_store()` first
([vector_db.py:177-179](project/starter/lib/vector_db.py#L177-L179)). So *constructing* the memory
object destroys the collection. Fine for a demo notebook that wants a clean slate each run;
catastrophic for the "agent learns from web searches" stand-out feature, where the entire point is
that data outlives the process.

This is the most dangerous bug in the file because **it fails silently**. B1 and B4 throw. This one
just quietly returns nothing, and you'll blame your retrieval logic.

**Impact.** Fails **silently, and it destroys data** — the worst combination in the file. Nothing
raises, nothing warns, and the observable behaviour ("memory is empty") is indistinguishable from a
retrieval bug, a threshold bug, or a write-back bug. So the cost isn't just the broken feature: it's
that you will go looking in three wrong places first. Concretely, the P7 stand-out feature is
100% dead, and because the wipe happens in `__init__`, *the act of inspecting the memory destroys
the evidence* — you cannot debug it by constructing a `LongTermMemory` and looking inside.

**The fix.** Use the non-destructive factory, and make wiping opt-in:

```python
def __init__(self, db: VectorStoreManager, reset: bool = False):
    self.vector_store = (
        db.create_store("long_term_memory", force=True) if reset
        else db.get_or_create_store("long_term_memory")
    )
```

`get_or_create_store` ([vector_db.py:191-196](project/starter/lib/vector_db.py#L191-L196)) already
exists and does exactly the right thing.

**Panel answer.** "Constructing `LongTermMemory` deleted the collection, so persistence could never
work. It failed silently rather than raising, which made it the hardest one to spot. I switched to
`get_or_create_store` and put the destructive path behind an explicit `reset` flag."

---

## B6 — `get_namespaces()` iterates a dict as if it were a list of records

**Hit in:** P7 · **Severity:** medium

**Symptom.** `TypeError: string indices must be integers` — or, once B4 is fixed, a confusing
failure inside a list comprehension.

**Root cause.** [memory.py:238-239](project/starter/lib/memory.py#L238-L239):

```python
results = self.vector_store.get()
namespaces = [r["metadatas"][0]["namespace"] for r in results]
```

**Why it happens.** Two stacked defects.

1. It calls the broken `.get()` from **B4**, so it raises before reaching the comprehension.
2. Even after B4 is fixed, the shape is wrong. Chroma's `get()` returns a **columnar** dict —
   `{"ids": [...], "documents": [...], "metadatas": [...]}` — not a list of row records. Iterating
   a dict yields its *keys*, so `r` is the string `"ids"`, and `r["metadatas"]` is a string index.

Columnar-vs-row is the thing to internalise: Chroma gives you parallel arrays, and you zip them
yourself. Note that `query()` returns the same columns wrapped in one extra list level (one entry
per query text) — which is why `search()` at
[memory.py:326-327](project/starter/lib/memory.py#L326-L327) correctly does `[0]` on each and this
method should not.

**Impact.** Fails **loudly**, and it's contained — `get_namespaces()` is a convenience/inspection
method, so `search()` and the write-back path work without it. Low stakes for the rubric. Its real
cost is that it's the natural first thing you reach for when debugging B5 ("let me just list what's
in memory"), and it throws — so the two bugs mask each other and you're now debugging your
debugger. Note the second defect survives fixing the first, which is why "I fixed B4" isn't enough.

**The fix.**

```python
results = self.vector_store.get()
metadatas = results.get("metadatas") or []
return list({m["namespace"] for m in metadatas if m and "namespace" in m})
```

**Panel answer.** "Chroma returns columnar results — parallel arrays keyed by field, not a list of
records. The original code iterated the dict, which yields keys. I also had to de-duplicate, since
the method promises *unique* namespaces and the original never did."

---

## N1 — `session_id` is missing from `AgentState` (this is NOT a bug)

**Discussed in:** P3 and P6 · **Severity:** none — but it's the best teaching example in the repo

**The trap.** [agents.py:11-16](project/starter/lib/agents.py#L11-L16) defines `AgentState` without
a `session_id` field. Yet [agents.py:55](project/starter/lib/agents.py#L55) reads
`state["session_id"]`, and every step returns it. Meanwhile
[state_machine.py:49-56](project/starter/lib/state_machine.py#L49-L56) filters each step's return
value against the schema:

```python
expected_fields = get_type_hints(state_schema)
updated = {**state}
for field, value in result.items():
    if field in expected_fields:
        updated[field] = value
```

Reading that, the obvious conclusion is: `session_id` gets dropped, and step two dies with a
`KeyError`. **It doesn't.** Do not "fix" this before understanding why.

**Why it actually works.** Line 53 is `updated = {**state}` — the machine starts from a *copy of the
existing state* and then overlays the filtered result. `Agent.invoke` seeds `session_id` into
`initial_state` at [agents.py:173](project/starter/lib/agents.py#L173), so it is present from the
first step onward and is simply carried forward by the spread. The filter only governs what a step
may **change**, not what the state may **contain**.

**The real consequence.** A step can never *modify* `session_id` — its return value is filtered
out every time. The three `"session_id": state["session_id"]` lines in the step functions are dead
code: they compute a value that is thrown away, and it happens to not matter because the spread
already preserved it.

**Impact.** Zero at runtime — nothing is broken. The cost is entirely in *your* time and
credibility: it is the most convincing-looking false positive in the repo, and if you "fix" it and
tell the panel you fixed a `KeyError`, a sharp examiner will ask you to reproduce it and you won't
be able to. Report it as a design wart with dead code, not a defect.

**What to do.** Adding `session_id: str` to the TypedDict is a reasonable tidy-up that makes the
returns meaningful and the schema honest. Make the change knowing it fixes a *documentation*
problem, not a crash.

**Panel answer.** "It looks like a `KeyError` waiting to happen, but `Step.run` spreads the previous
state before applying the filtered update, so a key seeded at entry survives. What the filter really
controls is which keys a step may *mutate* — so those `session_id` returns were dead code, not a
bug."

---

## N2 — `total_tokens` is missing from `initial_state` (load-bearing `.get()`)

**Discussed in:** P6 · **Severity:** none — but easy to break yourself

**The trap.** `AgentState` declares `total_tokens: int`, but `Agent.invoke` builds `initial_state`
without it ([agents.py:168-174](project/starter/lib/agents.py#L168-L174)). The only reason this
works is the defensive read at [agents.py:70](project/starter/lib/agents.py#L70):

```python
current_total = state.get("total_tokens", 0)
```

`TypedDict` is a **static** annotation with no runtime enforcement — Python will not populate a
declared key, and will not complain when it's absent.

**Impact.** Zero as shipped — but this is the one entry where *touching it* is the hazard. It's a
booby trap for a tidy-minded developer: the "cleanup" is a one-character change, looks strictly more
consistent than its neighbours, and breaks every single agent invocation immediately. A
self-inflicted P6 outage during Day 4.

**Why it matters to you.** If you "clean this up" to `state["total_tokens"]` for consistency with
the neighbouring lines, the first LLM step raises `KeyError`. Either leave the `.get()` alone, or
seed `"total_tokens": 0` in `initial_state` — do one, and know which.

**Panel answer.** "`TypedDict` is checked by the type checker, never at runtime, so a declared key
can be genuinely absent at execution time. That `.get()` with a default is load-bearing, not
defensive noise."

---

## Fix ledger

Mark each as you apply it. Copy the date into `PROGRESS.md` §3 with your rationale.

| ID | File | Phase | Applied | Explained |
| --- | --- | --- | --- | --- |
| B1 | `lib/vector_db.py` | P1 | [ ] | [ ] |
| B2 | `lib/vector_db.py` | P1 | [ ] | [ ] |
| B3 | `lib/vector_db.py` | P1 | ~~n/a~~ withdrawn | [ ] |
| B4 | `lib/vector_db.py` | P2 | [ ] | [ ] |
| B5 | `lib/memory.py` | P7 | [ ] | [ ] |
| B6 | `lib/memory.py` | P7 | [ ] | [ ] |
| N1 | `lib/agents.py` | P3/P6 | n/a | [ ] |
| N2 | `lib/agents.py` | P6 | n/a | [ ] |
