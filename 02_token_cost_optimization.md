# Token Cost Optimization — The Navigator Architecture

**This is your headline contribution. Know it cold.**

---

## 1. Interview Answer (2–3 minutes)

> Our code-generation agent had a knowledge problem. It writes Python against a proprietary
> in-house SDK, so the model has zero pretraining knowledge of it — every API, every convention has
> to come from context. The way that was solved originally was brute force: we had a single
> `STRATEGY_CREATION_GUIDE.md`, about 20,000 tokens, and it was concatenated into the code agent's
> system prompt at module load. Every single code request carried the entire guide.
>
> The waste was obvious once you looked at a real request. Someone asks "add a 2% trailing stop" and
> the model receives fifteen topics — pairs trading, timezone handling, multi-timeframe alignment,
> visualization panes — when it needs one. We were paying for about 19,000 tokens of irrelevant
> context on every call.
>
> What I built was a navigator plus on-demand retrieval. I wrote an offline compiler that reads the
> monolithic guide and decomposes it into fifteen self-contained topic pages, each with frontmatter
> tracing back to the exact line range in the source guide — so nothing is invented, and every page
> is auditable against ground truth. Then it generates an `INDEX.md`: a table of topic ID, aliases,
> and a "load when" trigger.
>
> At runtime, the agent's system prompt contains only the navigator — I parse INDEX.md with a regex
> and emit just the ID and the trigger column, about 350 tokens. And I added a tool, `read_sdk_page`,
> that takes a page ID and returns that page's markdown. So the agent reads a table of contents,
> decides it needs `exit-rules` and maybe `inline-state-machine`, pulls those two, and writes the code.
>
> The always-on floor went from 19,659 tokens to 792 — about 96% off. A typical call with two pages
> loaded is around 2,000 tokens, so roughly 89% off end to end. And critically, because the always-on
> block is static, it stays fully prompt-cacheable.
>
> The design decision I'd defend hardest is that I did *not* use a vector database. Fifteen
> well-separated topics in a closed, curated corpus is a routing problem, not a similarity problem.
> Exact-ID lookup can't retrieve the wrong chunk — there's no embedding drift, no threshold to tune,
> no recall failure mode. And I had a safety net: if `INDEX.md` is missing or unparseable, it falls
> back to injecting the full guide, so a bad compile degrades cost, never correctness.

---

## 2. What Problem Were We Solving?

### Business problem

Every code-generation turn cost real money, and the cost was roughly constant regardless of how
trivial the request was. Three consequences:

1. **Unit economics.** A free-tier user asking five simple questions burned the same input tokens as
   someone building a complex pairs-trading strategy. Token quota (`billing.check_token_quota`)
   throttled users on cost we were largely wasting.
2. **Latency.** ~20K tokens of prompt is time-to-first-token the trader watches as dead air.
3. **Headroom.** The guide only grows. Every new SDK feature made every unrelated request pricier.
   The architecture had a scaling term in the wrong place.

### Technical problem

The model needs grounding in a private SDK it has never seen. But "needs grounding" was being
translated as "needs *all* grounding, always." The mismatch:

| Request | Topics actually needed | Topics received |
|---|---|---|
| "Add a 2% trailing stop" | `inline-state-machine`, `exit-rules` | all 15 |
| "20/50 SMA crossover" | `template-structure`, `declarative-indicators` | all 15 |
| "Pairs trade NIFTY vs BANKNIFTY" | `pairs-trading`, `assets-and-intervals`, `timezone-handling` | all 15 |

### Why the existing approach wasn't enough

The wholesale-injection approach had exactly one virtue — it could never fail to include something
relevant. That's a real virtue and it's why it was built that way first. But:

- **It doesn't scale with the corpus.** Cost is O(guide size), not O(request complexity).
- **Long-context dilution is real.** Signal degrades when the relevant instruction is 5% of an
  enormous prompt. Removing 19K tokens of noise is a *quality* argument, not just a cost one.
- **Prompt caching helps but doesn't solve it.** Cached input tokens are cheaper, not free, and the
  cache only covers the static prefix.

---

## 3. Architecture

### Before

```
┌──────────────────────────────────────────────────────┐
│  code_expert system prompt (built at module import)  │
│                                                      │
│  PERSONA                              ~400 tok       │
│  CODE_GUARDRAILS                      ~500 tok       │
│  TOOL_BEHAVIOUR_*                     ~600 tok       │
│  ─────────────────────────────────────────────       │
│  SDK_GUIDE  ◄── entire STRATEGY_CREATION_GUIDE.md    │
│                                    19,659 tok        │
│                                                      │
│  Every request. Regardless of content.               │
└──────────────────────────────────────────────────────┘
```

### After

```
COMPILE TIME (offline, once, human-reviewed)
┌────────────────────────────────────────────────────────────┐
│  agent_kb/raw/STRATEGY_CREATION_GUIDE.md   [IMMUTABLE]     │
│                    │                                        │
│                    ▼                                        │
│    scripts/compile_sdk_wiki.py    (OpenAI Responses API)    │
│      --dry-run → --plan-only → review → --from-plan         │
│                    │                                        │
│      ┌─────────────┼──────────────┐                         │
│      ▼             ▼              ▼                         │
│  INDEX.md    topics/*.md    _token_report.md                │
│  (navigator)  (15 pages)    (before/after proof)            │
│               median 638 tok                                │
└────────────────────────────────────────────────────────────┘

RUN TIME
┌────────────────────────────────────────────────────────────┐
│  system_prompt.py :: _build_sdk_navigator()                │
│    reads INDEX.md, regex-extracts (page_id, load_when)     │
│    emits ~350-token navigator block                        │
│    on failure → returns "" → code_expert falls back to     │
│                              full SDK_GUIDE                │
└───────────────────────┬────────────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────────────┐
│  code_expert agent                                         │
│    _sdk_reference = SDK_NAVIGATOR if SDK_NAVIGATOR         │
│                     else "## SDK Reference\n\n" + SDK_GUIDE│
│    tools = [resolve_assets, read_head_file,                │
│             read_sdk_page, propose_files]                  │
└───────────────────────┬────────────────────────────────────┘
                        ▼
        "add a trailing stop" ──► agent reads navigator
                                  ├─ read_sdk_page("inline-state-machine")  → 882 tok
                                  └─ read_sdk_page("exit-rules")            → 638 tok
                                  ──► writes code ──► propose_files
```

### The retrieval call itself

```
agent                          read_sdk_page tool                 filesystem
  │                                    │                              │
  ├─ read_sdk_page("exit-rules") ─────►│                              │
  │                                    ├─ path = _WIKI_DIR/exit-rules.md
  │                                    ├─ os.path.exists()? ─────────►│
  │                                    │                              │
  │                              exists │◄─────────────────────────────┤
  │                                    ├─ read file                    │
  │◄── {"exists": true,                │                              │
  │     "page_id": "exit-rules",       │                              │
  │     "content": "..."}              │                              │
  │                                    │                              │
  │  ── miss path ──                   │                              │
  ├─ read_sdk_page("exitrules") ──────►│                              │
  │                            not exists                             │
  │◄── {"exists": false,               │                              │
  │     "available": [sorted list]}    │  ◄── self-correcting          │
```

That miss path is a small design detail worth mentioning in an interview: on a bad page ID the tool
returns the list of valid IDs rather than an error. The agent self-corrects on the next call without
a round trip to the user.

---

## 4. Deep Technical Explanation

### Component 1 — the immutable raw layer

`agent_kb/raw/STRATEGY_CREATION_GUIDE.md` is a byte-identical copy of the source guide, and the hard
rule in `SCHEMA.md` is: **the compiler reads it, never writes it.**

*Why it exists:* the compiled wiki is LLM-generated. LLM-generated documentation can hallucinate. If
you compile *in place*, you destroy your ground truth and can never audit drift. Keeping `raw/`
immutable means every generated page can be diffed against the exact lines it came from — forever.

*How it works:* every generated page carries `sources` frontmatter naming a line range in `raw/`.
Coverage tags (`[coverage: high|low]`) mark sections where the compiler was confident vs. where the
reader should fall back to the raw range.

### Component 2 — the offline compiler

`scripts/compile_sdk_wiki.py`, driven by `agent_kb/wiki/SCHEMA.md`, with a deliberately staged CLI:

```bash
uv run python scripts/compile_sdk_wiki.py --dry-run              # no API calls, no spend
uv run python scripts/compile_sdk_wiki.py --plan-only            # cheap; emits wiki/_plan.json
# ---- human reviews the decomposition ----
uv run python scripts/compile_sdk_wiki.py --from-plan wiki/_plan.json
```

*Why staged:* plan-then-generate separates the expensive, hard-to-reverse step (writing fifteen full
pages) from the cheap, high-leverage decision (what the fifteen pages should be). You catch a bad
decomposition for pennies instead of discovering it after a full compile. The `--dry-run` mode exists
so you can validate wiring with zero spend.

*Why offline and not at request time:* compilation is deterministic w.r.t. the guide, which changes
maybe monthly. Doing it per-request would be absurd. This is a build step, and treating it as one
means the output is reviewable in a PR — commit `dc3e340`.

*Why in-repo and not `llm-wiki-compiler`:* the off-the-shelf tool drives generation through an
interactive CLI — fine for a desktop session, wrong for a repeatable, reviewable repo step.

### Component 3 — the navigator (`_build_sdk_navigator`)

```python
rows = re.findall(r'\[\[topics/([^\]]+)\]\]\s*\|\s*[^|]+\|\s*([^|\n]+)', index)
```

That regex captures the **page id** and the **"load when" column**, and deliberately discards the
"also known as" column in between.

*Why discard aliases at runtime?* INDEX.md is a human-facing document — aliases help a person
skimming it. The model doesn't need synonyms to pick from fifteen options; it needs a trigger
condition. Dropping that column is a large fraction of the 792 → ~350 token reduction. **This is the
detail that separates "I read a blog post" from "I built this."**

The emitted block is a routing instruction plus a list:

```
## SDK Reference — Navigator
The full SDK guide is compiled into small pages. Call `read_sdk_page` to load
the page(s) relevant to this request, then write the code.
Load the fewest pages that cover the task — usually 1-2.

Available pages (use the id exactly):

- **template-structure** — building any strategy from scratch; the file shape
- **exit-rules** — fixed %/points stop or target
- ... (15 total)

If unsure, start with `template-structure` for a new strategy or
`best-practices` for a correctness check.
```

Three deliberate choices in that text: *"Load the fewest pages that cover the task — usually 1-2"*
budgets the agent explicitly. *"use the id exactly"* suppresses paraphrased IDs. The final fallback
line prevents dithering when the request is ambiguous.

### Component 4 — the retrieval tool

```python
@function_tool
async def read_sdk_page(ctx: RunContextWrapper[AgentContext], page_id: str) -> dict:
    path = os.path.join(_WIKI_DIR, f"{page_id}.md")
    if not os.path.exists(path):
        available = [f[:-3] for f in os.listdir(_WIKI_DIR) if f.endswith(".md")]
        return {"exists": False, "page_id": page_id, "available": sorted(available)}
    return {"exists": True, "page_id": page_id, "content": open(path, encoding="utf-8").read()}
```

Eleven lines. That's the entire "retrieval engine." Worth saying out loud in an interview — the
sophistication is in the *compile* step and the *routing* design, not in the lookup.

### Component 5 — graceful degradation

```python
_sdk_reference = SDK_NAVIGATOR if SDK_NAVIGATOR else "## SDK Reference\n\n" + SDK_GUIDE
```

`_build_sdk_navigator()` returns `""` on two failures — `INDEX.md` missing (`FileNotFoundError`), or
present but yielding zero regex matches. Both log a warning and fall back to the full guide.

*Why this matters:* it makes the optimization **strictly safe**. The worst case of a broken compile
is that you pay the old token cost. You never ship an agent with no SDK knowledge. When you're
removing 96% of a model's grounding, the fallback is not optional — and interviewers will ask what
happens when it breaks.

### Component 6 — prompt-cache alignment

The module docstring in `system_prompt.py` states the rule:

> **CRITICAL: Do NOT inject any dynamic content here** (no project name, no file content, no user
> data). Dynamic content goes in ephemeral input items.

The navigator is built once at module import and is byte-identical across every request. That keeps
the static prefix cacheable. Injecting anything per-user would invalidate the cache and cost more
than the navigator saves. Dynamic content rides in via `build_ephemeral_blocks()` and a
`session_input_callback` — never persisted, never in the cached prefix.

---

## 5. My Actual Contribution

**What existed**
- `agent_kb/STRATEGY_CREATION_GUIDE.md` — the 20K-token monolith.
- `system_prompt.py` with `_load_sdk_guide()` reading it at import.
- `code_expert.py` concatenating `"## SDK Reference\n\n" + SDK_GUIDE` into its instructions.
- The Agents-SDK tool mechanism (`@function_tool`).

**What I built**
- `agent_kb/wiki/SCHEMA.md` — the compile contract: page shape, `sources` frontmatter requirement,
  `raw/` immutability, the rule that persona and guardrails never enter the wiki.
- `scripts/compile_sdk_wiki.py` — the offline compiler with `--dry-run` / `--plan-only` /
  `--from-plan`.
- The compiled artifacts: `INDEX.md` navigator + 15 topic pages + `_token_report.md` + lint report.
- `quantjo_api/agent/tools/read_sdk_page.py` — the retrieval tool.
- `_build_sdk_navigator()` in `system_prompt.py` — the runtime navigator with fallback.
- The wiring in `code_expert.py`: `_TOOL_BEHAVIOUR_READ_SDK_PAGE`, the tool registration, and the
  conditional `_sdk_reference`.
- `"read_sdk_page": "Reading SDK reference"` in `TOOL_LABELS` so the UI shows it live.

**What changed**
- Always-on prompt on the code path: 19,659 → 792 tokens (~350 as emitted at runtime).
- Retrieval became explicit and observable — you can see in the trace which pages the agent chose.
- SDK guide edits now require a recompile step (real new operational cost — own it).

**Result**
- ~96% reduction in always-on tokens; ~89% on a typical two-page call.
- Lower time-to-first-token.
- The knowledge base can now grow without making unrelated requests more expensive — the scaling
  term moved from O(corpus) to O(pages actually needed).

---

## 6. Files and Code

| Path | Role |
|---|---|
| `agent_kb/README.md` | Layer explanation, compile runbook, hard rules |
| `agent_kb/wiki/SCHEMA.md` | Compile contract — compile-time only, never shipped to the model |
| `agent_kb/raw/STRATEGY_CREATION_GUIDE.md` | Immutable ground truth |
| `agent_kb/STRATEGY_CREATION_GUIDE.md` | Live guide; still loaded as fallback |
| `agent_kb/wiki/_tool/INDEX.md` | Generated navigator source |
| `agent_kb/wiki/_tool/topics/*.md` | 15 generated leaf pages |
| `agent_kb/wiki/_tool/_token_report.md` | Before/after measurement |
| `agent_kb/wiki/_tool/.compile-state.json` | Compile provenance |
| `scripts/compile_sdk_wiki.py` | The offline compiler |
| `quantjo_api/agent/system_prompt.py` | `_load_sdk_guide()`, `_build_sdk_navigator()`, `SDK_GUIDE`, `SDK_NAVIGATOR` |
| `quantjo_api/agent/tools/read_sdk_page.py` | `read_sdk_page()`, `_WIKI_DIR` |
| `quantjo_api/agent/agents/code_expert.py` | `_sdk_reference`, `_TOOL_BEHAVIOUR_READ_SDK_PAGE`, tool list |
| `quantjo_api/agent/agent_loop.py` | `TOOL_LABELS["read_sdk_page"]` |
| `quantjo_api/models/token_usage.py` | Per-turn token accounting |

**The 15 topics:** `template-structure`, `signal-generation`, `declarative-indicators`,
`manual-indicators`, `exit-rules`, `parameters`, `assets-and-intervals`, `timezone-handling`,
`visualization-panes`, `best-practices`, `pairs-trading`, `multi-timeframe`, `inline-state-machine`,
`condition-tracking`, `worked-examples`.

---

## 7. Design Decisions

### Why not a vector database?

The single most likely question. The answer:

| Factor | This corpus | When vectors win |
|---|---|---|
| Size | 1 guide, 15 topics | thousands of docs |
| Curation | hand-authored, stable | user-generated, churning |
| Separation | topics are cleanly disjoint | overlapping, fuzzy |
| Query shape | "which topic?" — routing | "what's semantically near?" |
| Failure mode | miss returns the valid ID list | silent wrong-chunk retrieval |
| Infra cost | zero | index, embed model, tuning, ops |

Fifteen options with human-written trigger conditions is a routing problem, and the model is
*already* a good router. Adding embeddings would introduce a similarity threshold to tune and a
recall failure mode to debug, in exchange for nothing. **Exact-ID lookup cannot retrieve the wrong
chunk.**

### Other decisions

| Decision | Rationale | Tradeoff |
|---|---|---|
| Compile offline, not per-request | Guide changes monthly; output is PR-reviewable | Guide edits need a recompile; wiki can go stale |
| Immutable `raw/` | LLM-generated docs must be auditable | Storage duplication (trivial) |
| Plan → review → generate | Catch bad decomposition cheaply | Slower compile loop |
| Strip aliases at runtime | Large token win; model doesn't need synonyms | Slightly weaker routing on odd phrasing |
| Fall back to full guide | Broken compile costs money, never correctness | Silent cost regression if unnoticed — needs alerting |
| Budget "usually 1-2 pages" | Bounds worst-case cost | Model may under-fetch on multi-topic requests |
| `chars/4` token heuristic | tiktoken was blocked by CodeArtifact pip auth | ±10–15% imprecision — **disclose this** |

---

## 8. Production Improvements

1. **Measure with real token counts.** The report is a `chars/4` heuristic because tiktoken couldn't
   be installed through CodeArtifact. `token_usage` already records actual per-turn usage — the honest
   next step is a query over that table segmented by pre/post rollout.
2. **Cache pages in memory.** `read_sdk_page` hits the filesystem every call. An LRU dict or reading
   all 15 pages into memory at import removes syscalls from the hot path.
3. **CI staleness gate.** Hash `raw/` and fail the build if `.compile-state.json` doesn't match —
   otherwise the wiki silently drifts from the guide.
4. **Instrument retrieval.** Log `(request → pages chosen)` and mine it. Pages never loaded are dead
   weight; pages always co-loaded should probably be merged.
5. **Alert on fallback.** Falling back to the full guide is currently a `logger.warning`. In
   production that's a 25x cost regression that should page someone.
6. **A/B the navigator.** Route 5% of traffic to full-guide injection and compare judge scores.
   Proves the optimization didn't cost quality — right now that's asserted, not measured.
7. **Two-tier retrieval at scale.** If the corpus reaches hundreds of pages, the navigator itself
   stops fitting. Then: embedding search to shortlist ~20 candidates, navigator to route among them.
8. **Semantic cache on generation.** Near-identical requests ("add 2% stop" vs "add a 2 percent
   stop") could reuse output entirely.

---

## 9. Common Interview Questions

1. Why was the prompt 20K tokens to begin with?
2. Walk me through what happens on a single code-generation request now.
3. Why didn't you use a vector database?
4. How did you measure the 20K → 350 reduction?
5. What if the agent picks the wrong page?
6. What if the agent needs a topic that doesn't exist as a page?
7. How do you keep the wiki in sync with the source guide?
8. Why compile offline instead of at request time?
9. What's the failure mode if `INDEX.md` is deleted in production?
10. How does this interact with prompt caching?
11. Why is `raw/` immutable?
12. Why plan-then-generate instead of one compile pass?
13. What did you lose by making this change?
14. How do you know quality didn't degrade?
15. Why 15 pages and not 5 or 50?
16. Why strip the aliases column?
17. What's the latency impact — you added a tool round trip?
18. How would this scale to a 500-page corpus?
19. What's the cost of a cache miss on the static prefix?
20. If you did it again, what would you change?

---

## 10. Strong Answers

**Q3 — Why no vector DB?**
> Because it's a routing problem, not a similarity problem. We had fifteen topics in a closed,
> hand-curated corpus with clean boundaries — declarative indicators versus manual indicators,
> ExitRule versus inline exits. Each page has a human-written "load when" trigger. The model reads
> fifteen triggers and picks one or two. Embedding search would add an index, an embedding model, a
> similarity threshold, and a retrieval failure mode — to solve a problem a table of contents solves
> exactly. And exact-ID lookup has a property embedding search never has: it cannot return the wrong
> chunk. If the corpus were thousands of user-generated documents I'd flip immediately, and I'd
> probably go two-tier — embeddings to shortlist twenty, then the navigator to route among them.

**Q4 — How did you measure it?**
> I have to caveat this one. The numbers in `_token_report.md` are a `chars/4` heuristic, not real
> tokenization — tiktoken was blocked by our CodeArtifact pip auth at the time. On that heuristic:
> 19,659 tokens before, 792 for the always-on navigator, about 1,430 with a median page loaded, about
> 2,068 for a typical two-page call. So ~96% off the floor, ~89% typical. The runtime navigator is
> smaller still — around 350 — because `_build_sdk_navigator` strips a column. For real numbers I'd
> query the `token_usage` table, which records actual per-turn input, output, and cached tokens. That
> was the documented plan and I'd want to close it before claiming the number precisely.

**Q5 — Wrong page?**
> Two safety nets. If the page ID doesn't exist, the tool returns `{"exists": false}` plus the sorted
> list of valid IDs, so the agent self-corrects on the next call — that's why the miss returns data
> instead of raising. If the ID is valid but the wrong topic, the agent can call again; the tool
> description explicitly permits multiple calls. And below that, `propose_files` validates the code
> against the SDK before the trader ever sees it, and the guardrails tell the agent to fix validation
> errors itself rather than surfacing them. So a bad retrieval degrades to an extra round trip, not
> to broken code reaching the user.

**Q7 — Staying in sync?**
> Honestly — that's the weakest part of what I built. Today it's a documented manual step: edit the
> guide, re-run the compiler, review, commit. `.compile-state.json` records provenance and every page
> has `sources` frontmatter pointing at line ranges in `raw/`, so drift is *detectable*. But nothing
> enforces it. What I'd add is a CI check that hashes `raw/` and fails the build if the compile state
> doesn't match — the same pattern as a lockfile check. That's a twenty-line CI job and it closes the
> gap properly.

**Q13 — What did you lose?**
> Three things. Operational complexity — there's now a build step that can be skipped, and a wiki
> that can go stale. A latency round trip — the agent makes one or two extra tool calls before it
> starts writing, though those are cheap relative to generation. And a class of failure that didn't
> exist before: the agent can now under-retrieve. With the full guide, everything was always present.
> I mitigated that with the "if unsure, start with template-structure" fallback line and by making
> multi-page reads explicitly allowed, but it's a real tradeoff, not a free win.

**Q17 — Latency?**
> Net positive, but it's genuinely a tradeoff. I added one or two sequential tool round trips —
> maybe a few hundred milliseconds each. But I removed about 19,000 tokens of prompt from every call,
> which cuts prefill time meaningfully and reduces time-to-first-token. And because the removed
> content was static and cacheable, the comparison isn't quite apples to apples — the honest framing
> is that I traded a large fixed prefill cost for a small variable round-trip cost, which wins for
> every request that doesn't need all fifteen topics. That's essentially all of them.

**Q20 — Do it again?**
> Install tiktoken properly and measure with real tokens from day one — the heuristic weakened the
> result. Wire the CI staleness gate at the same time as the compiler rather than deferring it. And
> A/B against the old path instead of running the regression harness only afterward, so I'd have a
> quality delta and a cost delta on the same axes.

---

## 11. Follow-up Questions

- "You said 350 tokens but the report says 792. Which is it?" *(Both — different artifacts. Explain
  the alias-stripping.)*
- "What stops the compiler from hallucinating an SDK API that doesn't exist?"
- "The pages average 638 tokens. What if one topic genuinely needs 5,000?"
- "How would you detect that the agent has started over-fetching pages?"
- "Does the navigator itself get prompt-cached? How do you know?"
- "You have boundary probes (B1–B6) that map to confusable topic pairs. Did the navigator make those
  worse?" *(Excellent question — the honest answer is that's exactly what the regression harness was
  built to answer.)*
- "What happens on a concurrent recompile while requests are in flight?"
- "If I gave you 10x the corpus tomorrow, what's the first thing that breaks?"

---

## 12. Senior Engineer Perspective

A senior engineer evaluates this on four axes:

**Did you measure, or assert?** Partially measured, with a disclosed heuristic and a documented plan
to get real numbers. Disclosing the heuristic is what makes it credible — hiding it is what would
sink you.

**Did you consider the failure mode?** Yes, and this is the strongest part. The fallback means the
worst case is cost, not correctness. Senior engineers care far more about the blast radius of your
change than its happy path.

**Did you understand the *shape* of the problem?** Choosing routing over similarity because the
corpus is small, closed, and cleanly separated — and being able to say precisely when that flips —
is the single most senior signal here. Reaching for a vector DB by default is the junior move.

**Did you close the loop?** Only partly. There's no CI staleness gate and no A/B on quality. Say so.
"Here's what I'd do next and why" is a stronger answer than pretending it's finished.

They will also note the **coupling to the filesystem**: `read_sdk_page` reads from a path relative to
the module. That works on a single deploy unit and breaks the moment the KB needs to be versioned
independently or served from object storage. Be ready to say that unprompted.

---

## 13. Common Mistakes

- **Calling it RAG without qualification.** It's retrieval, but there's no embedding, no vector
  store, no similarity. Say "navigator-based retrieval" and explain. If you say RAG and then can't
  answer "what distance metric?", you've lost the room.
- **Quoting 20K → 350 without the caveat.** Two different artifacts, and a `chars/4` heuristic.
  Quote it precisely and you sound rigorous. Quote it loosely and one follow-up exposes it.
- **Claiming a quality improvement you didn't measure.** Long-context dilution is a real, defensible
  *hypothesis*. It is not a result you have.
- **Forgetting the fallback.** It's the best design decision in the whole feature. Lead with it when
  asked about failure modes.
- **Not knowing why the regex drops a column.** That's the detail proving you wrote it.
- **Being defensive about the manual sync.** "That's the weakest part, here's the twenty-line fix" is
  a strong answer. Pretending it's solved is not.

---

## 14. Cheat Sheet

**The numbers**
- Before: **19,659** tokens always-on (`chars/4` heuristic)
- After (INDEX.md raw): **792** — ~96% off
- After (runtime navigator, aliases stripped): **~350**
- Typical call (nav + 2 pages): **~2,068** — ~89% off
- Heavy (nav + 3 biggest): **~3,337**
- 15 pages; median **638**; three biggest **811 / 852 / 882**

**The flow**
`raw/guide` → `compile_sdk_wiki.py` (dry-run → plan → review → generate) → `INDEX.md` + 15 pages →
`_build_sdk_navigator()` regex → ~350-tok block in `code_expert` prompt → agent calls
`read_sdk_page(id)` → filesystem read → code → `propose_files`

**The five talking points**
1. Routing, not similarity — that's why no vector DB.
2. `raw/` immutable → every generated page auditable to a line range.
3. Fallback to full guide → worst case is cost, never correctness.
4. Static navigator → prompt-cacheable; dynamic content goes in ephemeral blocks.
5. Miss returns the valid ID list → agent self-corrects without a user round trip.

**The three honest caveats**
- Token numbers are a `chars/4` heuristic (tiktoken blocked by CodeArtifact auth).
- Wiki↔guide sync is manual; no CI gate yet.
- No A/B proving quality held; the regression harness is the closest evidence.
