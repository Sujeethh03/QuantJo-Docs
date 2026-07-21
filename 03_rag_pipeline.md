# Retrieval Pipeline — What Was Actually Built vs. Vector RAG

> **Read the reality check in [index.md](index.md) before this file.** There is no ChromaDB, no
> embedding model, and no tree-sitter in this codebase. This file covers the retrieval system that
> *does* exist, then explains the vector-RAG design honestly as "what I'd build if the corpus grew" —
> which is a legitimate and impressive thing to discuss, as long as you never claim you shipped it.

---

## 1. Interview Answer (2–3 minutes)

> I built the retrieval layer that grounds our code-generation agent in a proprietary SDK. I want to
> be precise about what it is, because "RAG" gets used loosely: it's retrieval-augmented generation
> in the literal sense — the model retrieves documents and generates from them — but it is *not*
> embedding-based. There's no vector database.
>
> The pipeline has a compile phase and a runtime phase. At compile time, an offline script reads our
> single 20K-token SDK guide and decomposes it into fifteen self-contained topic pages. That's the
> chunking step — but it's semantic and human-reviewed rather than a sliding window, because the
> source document already has clean topical boundaries and arbitrary splits would cut a worked
> example in half. Each page carries frontmatter tracing back to a line range in an immutable copy of
> the original, so no page can drift from ground truth without it being detectable.
>
> At runtime, the agent's prompt contains a navigator — a table of page ID and a trigger condition,
> about 350 tokens. The agent reads that, decides which one or two pages it needs, and calls a tool
> called `read_sdk_page` with the exact page ID. That's an O(1) filesystem lookup, not a similarity
> search.
>
> The ranking step, if you want to call it that, is done by the model itself against human-written
> "load when" conditions rather than by cosine similarity over embeddings. And that was a deliberate
> choice: with fifteen well-separated topics in a closed, curated corpus, this is a routing problem.
> Exact-ID retrieval can't return the wrong chunk, which is a guarantee vector search structurally
> cannot make.
>
> I'd flip to embeddings if the corpus became large, user-generated, or fuzzy — and I can walk you
> through exactly what that would look like.

---

## 2. What Problem Were We Solving?

**Business.** Grounding cost scaled with corpus size instead of request complexity — see
[02](02_token_cost_optimization.md).

**Technical.** The model has no pretraining knowledge of `quantjo-sdk`. Every API, every convention
must arrive in context. The retrieval question is: *how do you get the right subset of a private
corpus into a prompt?*

**Why the existing approach wasn't enough.** "Send everything" is technically a correct retrieval
policy with perfect recall and terrible precision. It doesn't scale, and it dilutes signal.

**Why not immediately reach for vectors.** The standard reflex is chunk → embed → store → cosine
search. That's correct for large, heterogeneous, churning corpora. It is over-engineering for one
curated document with fifteen clean sections, and it introduces failure modes (threshold tuning,
chunk-boundary artifacts, silent wrong retrieval) that didn't exist before.

---

## 3. Architecture

### As built

```
COMPILE PHASE (offline, human-in-the-loop)
  raw/STRATEGY_CREATION_GUIDE.md   [immutable ground truth]
        │
        ▼  compile_sdk_wiki.py --plan-only
  wiki/_plan.json  ── HUMAN REVIEWS THE DECOMPOSITION ──┐
        │                                               │
        ▼  --from-plan                                   │
  ┌─────────────────────────────────────────────────────▼──┐
  │ topics/*.md  ×15   each with:                          │
  │   - sources: raw/ line range   ← auditability          │
  │   - [coverage: high|low] tags  ← confidence signal     │
  │ INDEX.md            navigator table                    │
  │ _lint_report.md     orphans / stale / contradictions   │
  │ _token_report.md    before/after                       │
  └────────────────────────────────────────────────────────┘

RUNTIME PHASE (per request)
  INDEX.md ──regex──► navigator (~350 tok, static, cacheable)
        │
        ▼ injected into code_expert system prompt
  ┌──────────────────────────────────────────────┐
  │  agent reads 15 (id, load_when) pairs        │
  │  ROUTING: model selects 1–2 ids              │
  └──────────────┬───────────────────────────────┘
                 ▼
  read_sdk_page("exit-rules")
        │
        ▼  os.path.join(_WIKI_DIR, f"{page_id}.md")
  exact-match file read → {"exists": true, "content": "..."}
        │
        ▼
  page content enters the agent's context as a tool result
        │
        ▼
  generation → propose_files → SDK validation
```

### What a vector pipeline would look like (NOT BUILT)

```
INGEST:  guide → chunker (AST-aware for code, recursive for prose)
                   → embedding model (e.g. text-embedding-3-small, 1536-d)
                   → ChromaDB collection, cosine distance
                   → metadata: {topic, line_range, chunk_type}

QUERY:   request → embed → top-k ANN search (k≈8)
                   → metadata filter (chunk_type, topic)
                   → rerank (cross-encoder) → top-3
                   → assemble context window → generate
```

The structural difference: in the built system, **the model routes**. In the vector system, **an
index routes and the model consumes**. The first is better when routing is easy and the corpus is
small; the second is necessary when routing is impossible to enumerate.

---

## 4. Deep Technical Explanation

### Chunking — semantic, not sliding-window

The decomposition unit is a **topic**, defined in `SCHEMA.md` and reviewed by a human in
`_plan.json` before any page is generated. The fifteen: `template-structure`, `signal-generation`,
`declarative-indicators`, `manual-indicators`, `exit-rules`, `parameters`, `assets-and-intervals`,
`timezone-handling`, `visualization-panes`, `best-practices`, `pairs-trading`, `multi-timeframe`,
`inline-state-machine`, `condition-tracking`, `worked-examples`.

*Why not fixed-size chunks?* Because the retrieval unit must be **independently useful**. A trader
asking about trailing stops needs the complete `inline-state-machine` explanation — the pattern, the
loop structure, and the worked example. A 512-token window would sever the example from the
explanation and the agent would generate half a pattern. Fixed-size chunking is a concession you make
when you can't afford semantic segmentation; here we could.

*Result:* median page 638 tokens, largest 882. Tight enough that loading two is cheap, complete
enough that one page fully answers one question.

### The `sources` frontmatter — auditability

Every page names the line range in `raw/` it was compiled from. This exists because **LLM-generated
documentation can hallucinate**. Without provenance you have a corpus you cannot trust and cannot
verify. With it, any page can be diffed against its source lines forever.

The `[coverage: high|low]` tags extend this: they tell the *reading agent* where the compiler was
confident, with instructions to fall back to the raw line ranges when coverage is low. That's an
unusual and good design touch — the retrieval layer signals its own uncertainty downstream.

### Ranking — model-as-router

There is no scoring function. Ranking is the model choosing among fifteen human-written trigger
conditions:

```
| exit-rules          | ExitRule, stop_loss, take_profit, fixed % exits | fixed %/points stop or target |
| inline-state-machine| trailing stop, bar-by-bar loop, dynamic exit    | trailing/anchored/path-dependent exits |
```

The "load when" column *is* the ranking signal, authored by a human who knows the domain. Compared to
cosine similarity this is: more precise (a human encoded the decision boundary), more debuggable (you
can read why), and more maintainable (fix a bad route by editing one line of English) — but it does
not generalize beyond enumerated options, and it doesn't scale past what fits in a prompt.

### Context injection

Retrieved content enters as a **tool result**, not as a system-prompt append. That matters:

- The system prompt stays static and prompt-cacheable.
- Retrieved pages are part of the turn's conversation, so the model sees them as evidence it
  actively fetched rather than standing instruction.
- The agent can fetch more mid-turn if the first page was insufficient.

Dynamic per-request content — active view, pending changeset state, rejection feedback — travels a
separate path entirely: `build_ephemeral_blocks()` in `ephemeral.py`, injected via
`make_session_input_callback()` and `RunConfig(session_input_callback=...)`. Those blocks are
**never persisted** to conversation history. Keeping ephemeral context out of both the cached prefix
and the durable history is the discipline that makes the caching work.

### The known seams — where retrieval is hardest

`INDEX.md` explicitly documents six "seams" where adjacent topics get confused, mapped to the B1–B6
boundary probes in the test harness:

| Seam | Confusable pair |
|---|---|
| B1 | declarative vs manual indicators |
| B2 | auto-pane vs explicit pane |
| B3 | ExitRule vs inline exit columns |
| B4 | vectorised vs session/time filter |
| B5 | single-asset vs secondary-asset alignment |
| B6 | vectorised vs inline state machine |

This is the most sophisticated idea in the retrieval design: **the retrieval layer knows its own
ambiguous boundaries, and the test suite probes exactly those boundaries.** Vector RAG systems
rarely have this — you find out about a bad boundary from a user bug report. Here it's a test case.

---

## 5. My Actual Contribution

**What existed** — one monolithic guide, injected wholesale. No retrieval of any kind.

**What I built** — the entire retrieval pipeline: the compile contract (`SCHEMA.md`), the compiler,
the fifteen-page decomposition, the navigator, the `read_sdk_page` tool, the fallback path, and the
seam documentation tying retrieval boundaries to test probes.

**What changed** — grounding went from "always everything" to "explicitly the relevant subset," and
retrieval choices became observable in the trace.

**Result** — ~96% fewer always-on tokens, auditable grounding, and retrieval boundaries that are
tested rather than assumed.

---

## 6. Files and Code

| Path | Role |
|---|---|
| `agent_kb/wiki/SCHEMA.md` | Compile contract: page shape, provenance rules, `raw/` immutability |
| `agent_kb/raw/STRATEGY_CREATION_GUIDE.md` | Immutable ground truth |
| `scripts/compile_sdk_wiki.py` | Chunking + generation (OpenAI Responses API) |
| `agent_kb/wiki/_tool/INDEX.md` | Navigator + documented seams |
| `agent_kb/wiki/_tool/topics/*.md` | 15 retrieval units |
| `agent_kb/wiki/_tool/_lint_report.md` | Orphans, stale refs, contradictions, oversize |
| `agent_kb/wiki/_tool/.compile-state.json` | Compile provenance |
| `quantjo_api/agent/tools/read_sdk_page.py` | `read_sdk_page()` — the retriever |
| `quantjo_api/agent/system_prompt.py` | `_build_sdk_navigator()` — index → prompt |
| `quantjo_api/agent/ephemeral.py` | `build_ephemeral_blocks()`, `make_session_input_callback()` |
| `quantjo-agent-tests/plugins/quantjo_plugin/cases/probes/` | B1–B6 seam probes |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| Semantic chunking by topic | Retrieval unit must be independently complete | Requires human review; not automatable |
| Exact-ID over similarity | Cannot retrieve wrong chunk; zero infra | Doesn't scale past prompt-sized navigators |
| Model-as-router | Human-authored decision boundaries, debuggable | Doesn't generalize beyond enumerated pages |
| Retrieved content as tool result | Preserves prompt cache; allows mid-turn re-fetch | Consumes conversation context |
| Provenance frontmatter | LLM-generated docs must be auditable | Compile-time discipline required |
| `[coverage]` tags | Retrieval signals its own uncertainty | Only useful if the agent honours them |
| Seams documented + probed | Ambiguity becomes a test, not a bug report | Requires maintaining probe pairs |

---

## 8. Production Improvements

- **In-memory page cache** — 15 pages, ~10K tokens total. Load at import, drop the filesystem from
  the hot path.
- **Serve the KB from S3 with versioning** so it deploys independently of the API image. Today
  `_WIKI_DIR` is a path relative to the module — the KB is welded to the deploy unit.
- **CI staleness gate** — hash `raw/`, fail if `.compile-state.json` disagrees.
- **Retrieval telemetry** — log (request, pages chosen, judge score). Pages never chosen are dead
  weight; pages always co-chosen should merge.
- **Two-tier retrieval at scale** — embeddings shortlist ~20, navigator routes among them. This is
  the honest answer to "what if it grows 10x."
- **Reranking** — only meaningful once you have a candidate set larger than the prompt.
- **Evaluate retrieval independently** — precision/recall@k on a labeled set of (request → correct
  page) pairs. Currently retrieval quality is only measured indirectly, through end-to-end judge
  scores.

---

## 9. Common Interview Questions

1. Is this RAG?
2. What's your chunking strategy and why?
3. Why not fixed-size chunks with overlap?
4. What's your embedding model? *(Trap — there isn't one. Say so.)*
5. How do you rank retrieved documents?
6. How do you handle a retrieval miss?
7. How does retrieved content enter the prompt?
8. Why is retrieved content a tool result and not a prompt append?
9. How do you know the compiled pages are faithful to the source?
10. What's a "seam" and why do you document them?
11. How would you add a vector store to this?
12. What's the recall of your retriever?
13. How do you evaluate retrieval quality separately from generation quality?
14. What happens if two topics both apply?
15. Why 15 pages?
16. How do you prevent the compiled wiki from drifting?
17. What's the latency of a retrieval call?
18. How does this interact with the conversation history and compaction?
19. When would you switch to embeddings?
20. What's the biggest weakness of this design?

---

## 10. Strong Answers

**Q1 — Is this RAG?**
> Literally yes, architecturally unusual. We retrieve documents and generate from them. But there's no
> embedding model and no vector store — retrieval is exact-ID lookup, and the routing decision is made
> by the model reading a table of contents rather than by an ANN index. I'd describe it as
> navigator-based retrieval. I'm deliberate about the terminology because if I said "RAG" and you
> asked about my distance metric, the honest answer is that there isn't one, and I'd rather lead with
> that than get caught by it.

**Q4 — What's your embedding model?**
> There isn't one — that's the design. Fifteen topics with human-written trigger conditions in a
> closed corpus is a routing problem the model solves directly. Adding embeddings would have added an
> index to maintain, a threshold to tune, and a silent-wrong-retrieval failure mode, in exchange for
> nothing measurable. If the corpus grew to thousands of documents or became user-generated, I'd add
> one — probably `text-embedding-3-small` for cost, with a cross-encoder rerank, and I'd keep the
> navigator as a second-stage router over the shortlist.

**Q12 — Recall?**
> I don't have a number, and I should be straight that this is a gap. Retrieval quality was only ever
> measured indirectly — through the end-to-end regression harness, where a bad retrieval shows up as
> a low `sdk_correctness` score from the judge. What I'd build is a labeled set of request → correct
> page pairs and measure precision and recall@1 and @2 directly, because right now a retrieval
> regression and a generation regression look identical in the reports.

**Q14 — Two topics apply?**
> That's the normal case, and it's designed for. The navigator tells the agent to load one or two
> pages and explicitly permits multiple `read_sdk_page` calls. The interesting version of your
> question is when two topics *conflict* rather than compose — declarative versus manual indicators,
> for instance, where doing both is wrong. `INDEX.md` documents six of those as "known seams," and
> each maps to a paired boundary probe in the test suite. Probe A should exhibit pattern A and *not*
> pattern B, and vice versa. So ambiguous boundaries are regression-tested rather than discovered in
> production.

**Q20 — Biggest weakness?**
> The navigator has to fit in the prompt. That's a hard ceiling — at maybe fifty or a hundred pages
> the routing table itself becomes the cost problem I set out to solve, and the model's ability to
> discriminate among that many trigger conditions degrades. The design is correct for a small curated
> corpus and structurally wrong for a large one. The migration path is two-tier: embeddings to
> shortlist, navigator to route. I'd want to instrument retrieval before making that call, so the
> switch is triggered by data rather than by a hunch.

---

## 11. Follow-up Questions

- "Your pages average 638 tokens. What's your policy when a topic legitimately needs 5,000?"
- "The compiler is an LLM. How do you know it didn't invent an SDK method?"
- "What does `[coverage: low]` actually cause the agent to do?"
- "If two pages contradict each other, what happens?"
- "How does retrieval interact with conversation compaction at 20 non-user items?"
- "You said the model routes. What's your evidence it routes *well*?"
- "Walk me through adding a 16th topic. What has to change?"

---

## 12. Senior Engineer Perspective

Three things a senior engineer will focus on:

**Provenance.** They'll immediately ask how you trust LLM-generated documentation. The `raw/`
immutability rule plus `sources` frontmatter is a genuinely strong answer — it's the same discipline
as generated code being traceable to its generator input.

**Evaluation of the retrieval layer in isolation.** This is your real gap. End-to-end scores conflate
retrieval failure with generation failure. A senior engineer will want them separated, because you
cannot fix what you cannot attribute. Name this before they do.

**The scaling cliff.** They'll want to know you understand the design has a ceiling, roughly where it
is, and what's on the other side. Knowing your architecture's expiry date is a senior trait.

They will also respect that you *didn't* build a vector database. Resisting the fashionable solution
when the simple one is correct is exactly what separates a senior engineer from a well-read junior —
but only if you can articulate the conditions under which you'd change your mind. Always pair the
"no vector DB" answer with the "here's when I'd flip" answer.

---

## 13. Common Mistakes

- **Claiming ChromaDB, embeddings, or tree-sitter.** They're not in the repo. One follow-up exposes
  it and everything else you said becomes suspect.
- **Saying "RAG" and stopping.** Qualify it immediately; you control the framing that way.
- **Describing chunking in generic terms.** "We chunked semantically" is a textbook phrase. "We split
  into fifteen topics because the retrieval unit has to be independently complete — a trailing-stop
  question needs the pattern *and* the worked example, and a 512-token window would sever them" is
  your answer.
- **Not knowing the ceiling.** "It scales fine" is wrong and detectable. It scales until the
  navigator doesn't fit.
- **Overstating measurement.** You measured tokens, not retrieval precision. Keep those separate.

---

## 14. Cheat Sheet

- **Not vector RAG.** No embeddings, no ANN, no vector store. Exact-ID lookup.
- **Chunking:** semantic, by topic, human-reviewed via `_plan.json`. 15 pages, median 638 tok.
- **Retrieval:** `read_sdk_page(page_id)` → `agent_kb/wiki/_tool/topics/{id}.md`. O(1).
- **Ranking:** the model, against human-written "load when" triggers in `INDEX.md`.
- **Injection:** tool result (keeps system prompt static and cacheable).
- **Ephemeral context:** separate path — `build_ephemeral_blocks()` + `session_input_callback`,
  never persisted.
- **Provenance:** `sources` frontmatter → line range in immutable `raw/`. `[coverage]` tags flag
  low confidence.
- **Seams:** 6 documented ambiguous boundaries (B1–B6) → paired probes in the test suite.
- **Miss handling:** returns `{"exists": false, "available": [...]}` → agent self-corrects.
- **Ceiling:** navigator must fit the prompt. Past ~50 pages → two-tier (embed shortlist → navigate).
- **Gap to admit:** no isolated retrieval precision/recall metric.
