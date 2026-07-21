# Mock Interview — 100 Questions

All questions are specific to this internship. Each carries a short answer key — enough to check
yourself, not a script to memorize. Where a question is a trap, it's marked **⚠ TRAP**.

**How to drill:** cover the answer column, speak the answer out loud, then compare. If you can't
answer in 30 seconds, go back to the linked file.

---

## A. Easy — Warm-up and Context (1–15)

| # | Question | Answer key |
|---|---|---|
| 1 | What did the product do? | Plain-English → algorithmic trading strategy code → backtest. Indian markets, NSE/NFO. Users are traders who understand markets but not Python. |
| 2 | What was your role? | Intern on the AI layer. Three things: token cost optimization, agent regression testing, real-time streaming. |
| 3 | What's the tech stack? | FastAPI, Python 3.12, SQLAlchemy 2.0 async, asyncpg, Pydantic v2, PostgreSQL 17, S3 (ap-south-1). Frontend Next.js 16/React 19/TS. OpenAI Agents SDK. |
| 4 | How big is the API surface? | ~54 REST endpoints + 1 WebSocket endpoint. |
| 5 | What is Jo? | The main conversational agent. Handles conversation, asset resolution, result analysis, and routes code work to a sub-agent. |
| 6 | What is Codex here? | `code_expert` — the code-generation sub-agent. `gpt-5.3-codex`, medium reasoning effort. Invoked as a tool, not a handoff. |
| 7 | What's an atomic changeset? | AI writes to S3 `pending/backtest.py`, user sees a diff, accepts → promoted to `head/backtest.py`. Reviewable, reversible AI edits. |
| 8 | Why UUID primary keys? | Prevents resource enumeration. Defense in depth, not the primary isolation control. |
| 9 | Where do strategy files live? | S3 at `{user_id}/projects/{project_id}/{head\|pending}/backtest.py`. Path convention — no artifact table. |
| 10 | What's in Postgres vs S3? | Postgres: queryable metadata (users, projects, chat, backtest summary metrics as JSONB). S3: files and multi-MB backtest artifacts. |
| 11 | How does auth work? | JWT — 24h access, 30d refresh tracked in DB for revocation. bcrypt via passlib. Cookie-based on the frontend. |
| 12 | What database migrations tool? | Alembic. `psycopg2-binary` is present only because Alembic is synchronous. |
| 13 | What package manager? | `uv` for the backend, `npm` for the frontend. SDK from private AWS CodeArtifact. |
| 14 | How many test cases in your harness? | 20 — 5 positive, 12 boundary probes (6 pairs), 3 QC traps. |
| 15 | What was the headline token number? | 19,659 → 792 always-on (~96%). ~350 as emitted at runtime after alias stripping. ~2,068 typical two-page call. |

---

## B. Medium — Token Optimization (16–30)

| # | Question | Answer key |
|---|---|---|
| 16 | Why was the prompt 20K tokens? | The whole `STRATEGY_CREATION_GUIDE.md` was concatenated into `code_expert`'s system prompt at module import — every request, regardless of topic. |
| 17 | What did you replace it with? | Offline compile into 15 topic pages + a navigator index; runtime `read_sdk_page(page_id)` tool for on-demand fetch. |
| 18 | **⚠ TRAP** What's your embedding model? | There isn't one. No vector DB, no embeddings. Exact-ID filesystem lookup. Explain why that was the right call. |
| 19 | Why no vector database? | Small, closed, curated corpus with 15 cleanly-separated topics — a routing problem, not a similarity problem. Exact-ID can't retrieve the wrong chunk. |
| 20 | When would you switch to embeddings? | Corpus grows large, becomes user-generated, or topics get fuzzy. Then two-tier: embed to shortlist ~20, navigator to route among them. |
| 21 | Why is `raw/` immutable? | The wiki is LLM-generated and can hallucinate. Immutable ground truth + `sources` line-range frontmatter makes every page auditable forever. |
| 22 | Why compile offline instead of per request? | The guide changes ~monthly; it's a build step. Offline output is reviewable in a PR. Per-request compilation would be absurd cost. |
| 23 | Why the `--plan-only` stage? | Separates the cheap high-leverage decision (what the 15 pages should be) from the expensive one (generating them). Catch a bad decomposition for pennies. |
| 24 | What happens if `INDEX.md` is missing? | `_build_sdk_navigator()` returns `""`, logs a warning, and `code_expert` falls back to injecting the full guide. Worst case is cost, never correctness. |
| 25 | Why does the regex drop a column? | It captures page ID + "load when" and discards "also known as". Aliases help humans skim; the model needs a trigger condition. Big share of 792 → ~350. |
| 26 | What if the agent requests a bad page ID? | Tool returns `{"exists": false, "available": [sorted ids]}` — the agent self-corrects on the next call without a user round trip. |
| 27 | How does this interact with prompt caching? | The navigator is static and built at import, so the prefix is byte-identical per request and cacheable. Dynamic content goes in ephemeral blocks instead. |
| 28 | How did you measure the reduction? | `chars/4` heuristic — tiktoken was blocked by CodeArtifact pip auth. **Disclose this.** Real numbers would come from the `token_usage` table. |
| 29 | What did you lose? | Operational complexity (a build step that can be skipped, a wiki that can go stale), one or two tool round trips, and a new under-retrieval failure mode. |
| 30 | How do you keep the wiki in sync? | Manual recompile today. Drift is detectable via `.compile-state.json` and `sources` frontmatter, but nothing enforces it. Fix: CI hash gate on `raw/`. |

---

## C. Medium — Regression Testing (31–48)

| # | Question | Answer key |
|---|---|---|
| 31 | Why can't you unit-test an LLM agent? | Nondeterministic output, non-local failures (one prompt word changes everything), continuous rather than binary quality, and multi-turn interaction. |
| 32 | What's your core reframing? | From "is the output correct" (unanswerable) to "did it get worse than baseline" (answerable via diffing). |
| 33 | What are the three layers? | L1 deterministic AST predicates (free, hard gate). L2 LLM judge, 4 dimensions (gates selectively, fails open). L3 boundary probe pairs (cross-case). |
| 34 | Why that ordering? | Cheapest and most certain first — L1 short-circuits so you don't pay for a judge call on code that was never going to pass. |
| 35 | What's a boundary probe? | A pair of cases across a confusable SDK concept. A must exhibit pattern A *and not* B; B the inverse. All four conditions → `crossover_clean`. |
| 36 | What does that catch? | **Over-correction** — you fix behaviour B and the agent starts applying it everywhere, breaking A. Per-case testing misses it because A still passes its own assertions. |
| 37 | How many boundaries? | Six — B1 declarative vs manual indicators, B2 auto vs explicit pane, B3 ExitRule vs inline exits, B4 vectorised vs session filter, B5 single vs multi-asset alignment, B6 vectorised vs inline state machine. |
| 38 | What's the user simulator? | An LLM (gpt-4o, T=0) playing the human with a persona and goal. Emits `[DONE]` when its goal is met. |
| 39 | Why can't it see tool calls? | `_strip_tool_internals` filters them. A real user sees only text — if the simulator reacted to tool calls you'd be testing a fictional interaction. |
| 40 | Why the `[DONE]` sentinel? | Without it every conversation runs to `max_turns`, burning tokens and polluting the transcript the judge then scores. |
| 41 | How do you handle nondeterminism? | k repetitions with majority vote. `half = (n+1)//2`. Split verdict → **`flaky`**, not a coin-flip pass/fail. |
| 42 | Why a third verdict? | Instability is information. An agent correct 60% of the time is a different product from one correct 100% of the time. Collapsing that hides it. |
| 43 | How is a regression detected if the test passes? | Baseline diff on required judge dimensions — a score drop ≥ 0.15 is blocking even when the case still passes the 0.7 threshold. |
| 44 | Why is the baseline "last state" not "last green"? | If you only save green baselines, a legitimately-changed expectation leaves you diffing against an ancient run forever and the diff becomes noise. |
| 45 | Why a separate repository? | It tests the **deployed system**, coupled only over the network. If it imported the app, it would test the code in the repo — the very thing that might be wrong. |
| 46 | What's a waiver? | `{dimension, reason, expires}` — suppresses a known-bad dimension with documentation and a date, instead of deleting the test to get green. |
| 47 | Did you wire it into CI? | No. It's CI-*ready* — a pytest plugin and CLI with `--repetitions`, plus `get_blocking_diffs` as the gate function. The workflow file was the next step. |
| 48 | What's `aggregate_tokens` in your reports? | Always zero — a real gap. The WS client doesn't consume the server's `token_usage` event. Small, fixable. |

---

## D. Medium — AST and Judge (49–60)

| # | Question | Answer key |
|---|---|---|
| 49 | **⚠ TRAP** Did you use tree-sitter? | No — Python stdlib `ast`. And it was the better choice: zero deps, and it's the same parser that will actually run the code. |
| 50 | Why AST over regex? | Regex matches text — it hits docstrings, comments, and string literals. AST matches language constructs. Plus free syntax validation and nested `Attribute` chain resolution. |
| 51 | Give a concrete regex failure. | `r"class\s+\w+\(Strategy\)"` matches inside a docstring, a comment, and `example = "class Demo(Strategy)"`. The AST walk sees a `ClassDef` or it doesn't. |
| 52 | What can AST *not* do? | Anything semantic. It sees a loop; it can't say the loop is correct or that the strategy matches the trader's intent. That's why Layer 2 exists. |
| 53 | Why does `parses_to_python` always return True? | `run_all_predicates` parses first and short-circuits on `SyntaxError`, so no predicate runs unless parsing succeeded. It's a name letting a case declare the requirement in YAML. |
| 54 | What's the predicate contract? | `fn(tree, *, source, **kwargs) -> (bool, str)`. Evidence string is mandatory — a failure has to explain itself. |
| 55 | Why a plugin for SDK predicates? | Keeps the core harness agent-agnostic. 5 generic predicates in core, 19 SDK-specific via a `pytest11` entry point. |
| 56 | What are the judge dimensions? | V2: `intent_capture`, `sdk_correctness`, `exit_completeness`, `conversation_quality` (advisory, transcript-only). |
| 57 | What are the thresholds? | Pass ≥ 0.70 · fail < 0.50 · else `unsure` · regression Δ ≥ 0.15. |
| 58 | Why re-derive the verdict from the score? | Thresholds are a mechanical rule; models apply mechanical rules inconsistently but produce reasonably calibrated numbers. Never let the model do arithmetic you can do. |
| 59 | What's shadow mode? | Runs both judge backends; OpenAI gates pass/fail, DeepEval rides along as `shadow_verdict`. Safe evaluator migration — shadow traffic applied to evaluation. |
| 60 | Why does the judge fail open? | An API timeout says nothing about agent quality, and a gate that cries wolf gets ignored. L1 still hard-gates. Residual risk: silent loss of semantic gating if unmonitored. |

---

## E. Medium — Streaming and Backend (61–74)

| # | Question | Answer key |
|---|---|---|
| 61 | Why WebSocket over HTTP? | 30–60s turns need server-push progress, and the channel is bidirectional — accept/reject changeset, cancel, queued messages. |
| 62 | **⚠ Sharpest question** Why not SSE? | SSE is unidirectional. You'd need SSE + a REST side-channel + correlation logic. For read-only streaming I'd have picked SSE. |
| 63 | How does the streaming work mechanically? | `Runner.run_streamed()` → `async for event in result.stream_events()` → filter `run_item_stream_event` → `_handle_stream_event`. |
| 64 | What are the three event branches? | `tool_called` → `TOOL_LABELS` lookup. `tool_output` → error/success status + harvest `view`. `reasoning_item_created` → "Thinking" heartbeat. |
| 65 | Why the reasoning heartbeat? | Codex reasons for long stretches with no tool calls. Without it the status freezes and users think it hung. |
| 66 | What happens on an unknown tool name? | Nothing is emitted — the prior label persists. Never show a raw identifier like `resolve_assets_v2` to a non-technical trader. Fail quiet, not ugly. |
| 67 | Why aren't status updates chat messages? | They were, and it was a bug: history pollution, re-sent to the model every turn (repeated token cost), early compaction trigger, spam on reload. Fixed by `emit_status`. |
| 68 | How is a turn signalled complete? | `status` with `label: null`. Implicit — a known weakness. An explicit `turn_complete` frame with `message_id` would be better. |
| 69 | How does WebSocket auth work? | JWT in the first frame, not the query string — URLs leak into logs, history, and `Referer` headers. |
| 70 | Two messages sent quickly? | Single-slot buffer, latest-wins, emits `message_queued`. Users want the last message answered; earlier ones are usually corrections. |
| 71 | Why one DB session per turn? | `AsyncSession` isn't coroutine-safe. The turn is a background task racing the receive loop; sharing corrupts the identity map → "Instance is not persistent". |
| 72 | Why `db.get` instead of `db.refresh`? | `refresh` raises on an instance detached by a rollback inside a tool; `get` is identity-map-aware and returns it. |
| 73 | Why is `get_db` a generator? | Code after `yield` (the `async with` exit) is teardown that runs even if the handler raises — guaranteed session cleanup. |
| 74 | What does async actually buy here? | ~95% of a turn is I/O wait. A sync framework would pin a worker per turn. Note the footgun: one blocking call stalls the entire loop. |

---

## F. Hard — Architecture and Depth (75–86)

| # | Question | Answer key |
|---|---|---|
| 75 | **⚠ TRAP** Did you use LangGraph? | No — OpenAI Agents SDK. `Agent`, `Runner.run_streamed`, `@function_tool`, `RunConfig`. No graph, no nodes, no edges. |
| 76 | Sub-agent as tool vs handoff — why? | Control never leaves Jo. Handoffs need hand-back rules in every specialist prompt and produce "stuck in the wrong agent" failures. Tool call = function call. |
| 77 | How is context isolated for Codex? | Fresh ephemeral session per invocation containing only HEAD `backtest.py`, the trader's intent, and optional rejection feedback. It doesn't inherit chat history. |
| 78 | Static vs ephemeral context — explain. | System prompt is static and cacheable (no dynamic content, by rule). Per-turn context goes via `build_ephemeral_blocks` + `session_input_callback` and is **never persisted**. |
| 79 | Why never persist ephemeral context? | It goes stale ("user is viewing backtest results" is wrong three turns later) and it would cost tokens on every subsequent turn forever. |
| 80 | How is conversation history bounded? | `OpenAIResponsesCompactionSession` with `gpt-5-nano`, manual trigger at 20 **non-user** items, run in a background task after the response. |
| 81 | Why count non-user items only? | User messages are the signal you least want to lose. Tool calls and assistant messages are what bloat context. |
| 82 | How do you prevent duplicate agent runs? | Optimistic lock on `message_id`; a duplicate returns `ai_state.last_response` with `cached: true`. Reconnect-retries don't double-charge tokens. |
| 83 | Where does token usage come from? | `result.context_wrapper.usage` — `RunResultStreaming` has no `.usage`. It accumulates the nested Codex sub-run. `input_tokens_details.cached_tokens` measures cache hits. |
| 84 | What's `max_turns=10` for? | Bounds the tool-calling loop. An agent that never converges is unbounded cost. |
| 85 | Is the Postgres/S3 boundary transactional? | No. S3 write + failed commit leaves an orphaned pending file. Bounded by the path convention (next proposal overwrites). Fix: outbox pattern. |
| 86 | What is `STRATEGY_INTAKE` and when is it active? | A soft rule stopping code generation on a new project's first message — read the idea back, name gaps, recommend an MVP, ask **one** question. Activated by the `is_new_project` ephemeral marker. |

---

## G. Hard — Scalability, Security, Ops (87–94)

| # | Question | Answer key |
|---|---|---|
| 87 | What breaks first at scale? | In-process `ConnectionManager` (`Dict[UUID, Set[ConnectionState]]`) — a second instance can't reach connections on the first. Fix: Redis pub/sub + sticky sessions. |
| 88 | Second and third? | (2) Synchronous backtests pin the process. (3) DB pool — each turn holds a session 30–60s, so 20 connections ≈ 20 concurrent turns. |
| 89 | Best fix for the pool problem? | Not a bigger pool — **release the session across the LLM call**. The turn only needs the DB at the start and end; the middle is pure network wait. |
| 90 | What's the ceiling you can't engineer around? | OpenAI rate limits and cost. This is why token optimization *is* a scalability contribution — fewer tokens per request means more concurrent requests under a fixed TPM limit. |
| 91 | You execute LLM-generated code. Is that safe? | Not sufficiently. Two denylist checks (`eval`, `exec`, `__import__`, `os.system`, `subprocess.*`). Denylists are bypassable. Real control = sandboxed execution + import allowlist. |
| 92 | How do you mitigate prompt injection? | Architecturally: narrow tools, `AgentContext` scoped to `user_id`/`project_id` (not parameterized by model output), human accept gate. No sanitization or detection. The accept gate isn't a control against a *malicious* user. |
| 93 | What's wrong with your rate limiting? | In-memory `defaultdict` — resets on deploy, and N instances give a user N× the limit. The docstring says so. Fix: Redis atomic counters. |
| 94 | **⚠ TRAP** Walk me through your CI/CD. | There are no workflow files. Review discipline exists (CODEOWNERS, PR template/process); automation doesn't. Say so — it's trivially checkable. |

---

## H. Behavioural and Reflective (95–100)

| # | Question | Answer key |
|---|---|---|
| 95 | Tell me about a hard bug. | The `AsyncSession` identity-map race. Non-deterministic "Instance is not persistent" errors; correlated with messages sent mid-turn; root cause was a background task and the receive loop sharing one session. Fix: session per turn. Lesson: session scope follows your concurrency model, not convention. |
| 96 | Tell me about improving performance or cost. | The navigator. 20K → 792 always-on, ~96%. Key decisions: routing over similarity, immutable ground truth, fallback so worst case is cost not correctness. Reflection: measure with real tokens; add the CI staleness gate. |
| 97 | What are you proudest of? | Boundary probe pairs. ~80 lines, but they catch over-correction — a failure mode conventional per-case testing structurally cannot see, because the case you broke still passes its own assertions. |
| 98 | What's the weakest part of your work? | Verification of my own claims. Token numbers are a heuristic; the judge is uncalibrated against human labels; I never A/B'd the navigator against the old path. I was better at building than at proving. |
| 99 | What would you do differently? | Build measurement before optimization on all three projects. And ship the CI staleness gate alongside the compiler instead of deferring it — deferred gates accumulate invisible risk. |
| 100 | What did you learn? | That the hard part of production LLM systems isn't getting the model to work — it's knowing whether it still works tomorrow. Everything valuable I built was about making a nondeterministic system observable, testable, and affordable. |

---

## Drill Plan

**Night before, 45 minutes:**
1. Section A end to end (5 min) — these must be instant.
2. Every **⚠ TRAP** question: 18, 49, 75, 94 (5 min). Getting caught on one of these is the single
   biggest risk.
3. Questions 96, 95, 97 out loud, full 2-minute versions (15 min). These are your stories.
4. Section G (10 min) — the senior-level differentiators.
5. Questions 98–100 (10 min) — reflective answers are what interviewers remember.

**If you only have 10 minutes:** the four traps, plus 15, 36, 43, 71, 87, 90.

**The three numbers to never get wrong:** 19,659 → 792 (~96%) · 20 cases (5 + 12 + 3) ·
0.70 / 0.50 / 0.15.

**The one sentence to open with:** *"The platform existed. I owned three pieces of the AI layer:
making code generation cheaper, making it testable, and making it visible."*
