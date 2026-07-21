# My Actual Contributions

**Read this file first and last. It is the ground truth of what you can defend under pressure.**

---

## 1. Interview Answer (2–3 minutes)

> I interned on the AI layer of an algorithmic trading platform — the agent that turns a trader's
> plain-English description into working strategy code. I owned three pieces.
>
> The first was token cost. Our code-generation agent had a twenty-thousand-token SDK guide
> concatenated into its system prompt on every single call, whether the request needed one topic or
> fifteen. I built an offline compiler that decomposes that guide into fifteen self-contained topic
> pages with provenance back to the original, generates a navigator index, and a runtime tool that
> lets the agent fetch pages by ID. The always-on prompt went from about twenty thousand tokens to
> under eight hundred — around ninety-six percent — with a fallback to the old behaviour if anything
> in the compile is broken, so the worst case is cost rather than correctness.
>
> The second was testing. There were no automated tests for the agent at all, and prompt changes had
> unbounded blast radius. I built a standalone regression harness — twenty multi-turn conversations
> driven by an LLM user simulator against a live server, scored in three layers: deterministic AST
> assertions, an LLM judge across four dimensions, and boundary probe pairs that catch over-correction.
> Each case runs k times with a majority vote so instability shows up as "flaky" rather than randomly
> passing, and every run diffs against a stored baseline so a quality drop is blocking even when the
> test technically still passes.
>
> The third was real-time streaming. Agent turns take thirty to sixty seconds and the UI showed a
> spinner. I built the event dispatch that consumes the Agents SDK's streaming interface and pushes
> tool-level progress over the WebSocket, plus the label mapping that turns internal tool names into
> user-facing phrases. The important part was making those status events ephemeral rather than chat
> messages — an earlier version persisted them, which polluted conversation history and got re-sent to
> the model every turn.
>
> Underneath all three, the thing I'd point to as the most valuable habit is that each one has an
> explicit failure story: the navigator falls back, the judge fails open, unknown tools emit nothing.

---

## 2. What Problem Were We Solving?

Three problems, all in the AI layer, all sharing a root cause: **the agent was expensive, unverifiable,
and opaque.**

| Problem | Symptom | My work |
|---|---|---|
| Expensive | ~20K tokens on every code request regardless of complexity | Navigator + retrieval |
| Unverifiable | A one-line prompt change could silently break every user | Regression harness |
| Opaque | 30–60s of spinner; users assumed it had hung | Streaming progress |

---

## 3. Architecture — where my work sits

```
┌─────────────────────────────────────────────────────────────────┐
│  BROWSER                                                        │
│    ◄──── status: "Reading SDK reference"  ─────┐  [CONTRIBUTION 3]│
└──────────────────────┬─────────────────────────┼────────────────┘
                       │ WebSocket               │
┌──────────────────────▼─────────────────────────┼────────────────┐
│  FASTAPI                                       │                │
│                                                │                │
│   agent_loop.py                                │                │
│     _handle_stream_event() ────────────────────┘  [CONTRIB 3]   │
│     TOOL_LABELS                                                 │
│         │                                                       │
│         ▼                                                       │
│   main_agent → propose_code_change → code_expert                │
│                                        │                        │
│                          ┌─────────────▼──────────────┐         │
│                          │ SDK_NAVIGATOR (~350 tok)   │ [C 1]   │
│                          │ read_sdk_page(page_id) ────┼──┐      │
│                          └────────────────────────────┘  │      │
└──────────────────────────────────────────────────────────┼──────┘
                                                           ▼
                              ┌──────────────────────────────────┐
                              │  agent_kb/            [CONTRIB 1]│
                              │   raw/  (immutable ground truth) │
                              │   wiki/_tool/INDEX.md            │
                              │   wiki/_tool/topics/*.md  ×15    │
                              │   _token_report.md               │
                              │  scripts/compile_sdk_wiki.py     │
                              └──────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  quantjo-agent-tests   (separate repo)          [CONTRIBUTION 2]│
│    20 cases → WS client → user simulator → agent                │
│    Layer 1 AST · Layer 2 LLM judge · Layer 3 boundary pairs      │
│    majority vote → baseline diff → reports                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Deep Technical Explanation

### Contribution 1 — Token cost optimization

**Design.** Chose navigator-based routing over vector retrieval, because the corpus is small, closed,
curated, and cleanly separated — a routing problem, not a similarity problem. Designed the two-layer
`raw/` → `wiki/` structure with mandatory provenance frontmatter so LLM-generated pages stay auditable.
Designed a staged compile (`--dry-run` → `--plan-only` → human review → `--from-plan`) so a bad
decomposition is caught for pennies.

**Coding.** `agent_kb/wiki/SCHEMA.md` (the compile contract), `scripts/compile_sdk_wiki.py`,
`_build_sdk_navigator()` in `system_prompt.py`, `quantjo_api/agent/tools/read_sdk_page.py`, and the
wiring in `code_expert.py` (`_sdk_reference`, `_TOOL_BEHAVIOUR_READ_SDK_PAGE`, tool registration).

**Debugging / hardening.** The regex that parses `INDEX.md` deliberately captures only the page ID and
the "load when" column, discarding aliases — a large share of the 792 → ~350 token reduction. The
retrieval miss path returns the sorted list of valid IDs so the agent self-corrects instead of failing.
The navigator returns `""` on either failure mode (missing file, unparseable rows), and `code_expert`
falls back to the full guide.

**Testing.** Validated against the regression harness — that's what the KB README means by "Re-run
`quantjo-agent-tests`, diff against baseline."

**Optimization.** 19,659 → 792 always-on (~96%); ~2,068 on a typical two-page call (~89%). Kept the
navigator static so the prompt prefix stays cacheable.

**Honest caveats:** token numbers are a `chars/4` heuristic (tiktoken was blocked by CodeArtifact pip
auth); wiki↔guide sync is a manual step with no CI gate; no A/B proving quality held.

### Contribution 2 — Agentic regression harness

**Design.** Reframed the problem from "is the output correct" — unanswerable — to "did it get worse
than last time," which is answerable via baseline diffing. Chose a three-layer scoring architecture
ordered cheapest-and-most-certain first. Designed boundary probe pairs to catch over-correction, which
per-case testing structurally cannot. Chose a separate repo coupled only over the network so it tests
the deployed system rather than the source.

**Coding.** The entire package: `client/websocket_client.py` and `protocol.py`;
`harness/user_simulator.py`, `runner.py`, `ast_utils.py`, `case_loader.py`, `baseline.py`,
`reporter.py`, `scorecard.py`; `harness/judge/` (prompt, client, schema, deepeval client);
`assertions/layer1.py`, `layer2.py`, `layer3.py`; `cli.py`, `config.py`, `plugins.py`,
`pytest_plugin.py`; `plugins/quantjo_plugin/` with 19 SDK predicates and the B1–B6 registrations; and
20 case YAMLs.

**Debugging.** The user simulator's prompt is a list of patched failure modes — "NEVER ask a
question," "NEVER repeat, paraphrase, or confirm your own previous statement," "Do NOT reveal that you
are a simulator." Each line exists because the simulator drifted into assistant behaviour, echoed
itself, or broke character. `_strip_tool_internals` exists because the simulator was reacting to tool
calls a real user would never see. The `[DONE]` sentinel exists because every conversation was running
to `max_turns`, burning tokens and polluting transcripts.

**Testing.** Meta-level: a `no_llm_required` pytest marker for harness-infrastructure tests that don't
need a live model.

**Optimization.** Layer ordering short-circuits before paying for judge calls; `judge=off` for fast
iteration; the `[DONE]` sentinel ends conversations early.

**Honest caveats:** no judge calibration against human labels; `aggregate_tokens` is initialized but
never populated because the WS client ignores the `token_usage` event; the 0.15 regression threshold
is intuition, not measured variance; execution is serial; never wired into a CI gate.

### Contribution 3 — Real-time streaming

**Design.** Separated transient UI state from durable conversation state — the central insight.
Decided unknown tool names emit nothing rather than falling back to raw identifiers.

**Coding.** `_handle_stream_event()` in `agent_loop.py`, `TOOL_LABELS`, the `emit_message` →
`emit_status` migration with `emit_status(None)` as idle/turn-complete, the `reasoning_item_created`
heartbeat, and `view`-directive harvesting via `tool_extras`.

**Debugging.** Two real bugs here, and both are good stories:

1. **History pollution.** Tool progress emitted as chat messages persisted into conversation history,
   got re-sent to the model every turn (paying tokens repeatedly), pushed conversations past the
   20-item compaction threshold early, and showed as spam on reload.
2. **The `AsyncSession` identity-map race** — see below. This is your best debugging story.

**Testing.** Consuming this protocol from `AgentWebSocketClient` in the harness is what forced the
turn-complete semantics to be pinned down precisely — building the automated client surfaced the
protocol's ambiguity.

---

## 5. The Three Stories You Should Have Ready

Interviewers ask for stories, not architectures. Have these three rehearsed.

### Story A — "Tell me about a time you improved performance or cost"

**Situation.** Our code-generation agent injected a 20K-token SDK guide into every request.
**Task.** Cut cost without losing grounding quality.
**Action.** Compiled the guide into 15 topic pages with provenance, built a ~350-token navigator and
an exact-ID retrieval tool, and added a fallback to the old behaviour on any compile failure.
**Result.** ~96% reduction in always-on tokens; ~89% on a typical call; prompt stayed cacheable.
**Reflection.** I'd install tiktoken and measure with real tokens rather than a `chars/4` heuristic,
and I'd wire a CI staleness check so the wiki can't silently drift from the guide.

### Story B — "Tell me about a hard bug"

**Situation.** Intermittent "Instance is not persistent" SQLAlchemy errors on the WebSocket path.
Non-deterministic — they wouldn't reproduce on demand.
**Task.** Find and fix a race that only appeared under real usage.
**Action.** The pattern was that failures correlated with users sending a message while a turn was
already running. The agent turn runs as a background `asyncio.Task` so the receive loop can stay
responsive for cancellation and queued messages — and both were using the same connection-level
`AsyncSession`. `AsyncSession` isn't safe for concurrent coroutine use: two coroutines interleaving on
it corrupt SQLAlchemy's identity map, and the symptom is an object the session no longer believes is
persistent. The fix was giving each turn its own `AsyncSessionLocal()` and a fresh `AgentContext`
bound to it, so tools within a turn share one transaction and separate turns share nothing. There was
a related detail — using `db.get(Project, id)` instead of `db.refresh(project)`, because `refresh`
raises on an instance detached by a rollback inside a tool while `get` is identity-map-aware.
**Result.** The class of errors disappeared.
**Reflection.** The lesson I took is that a database session is a unit of work, and session scope has
to follow your concurrency model rather than your convention. "One session per request" is a heuristic,
not a rule, and the moment you introduce background tasks it's the wrong one.

### Story C — "Tell me about something you built that others use / a quality problem you solved"

**Situation.** No automated tests on the agent. A one-line prompt change could silently degrade output
for every user, and we'd find out from a support ticket.
**Task.** Make agent quality verifiable despite nondeterministic output.
**Action.** Built a 20-case multi-turn harness with three scoring layers. The design insight was
reframing from "is it correct" to "did it get worse," so the gate is a baseline diff rather than an
absolute grade. The piece I'm proudest of is Layer 3 — boundary probe pairs. For each of six confusable
SDK concepts, probe A must exhibit pattern A and must *not* exhibit pattern B, and vice versa. That
catches over-correction, which is the characteristic failure of prompt engineering and which per-case
testing structurally cannot see, because the case you broke still passes its own assertions.
**Result.** Prompt and agent changes became verifiable, with per-dimension scores tracked across runs.
**Reflection.** The gap I'd close is calibrating the judge against human labels. Without that, it
detects change well but can't certify correctness — and I'd rather state that limit than oversell it.

---

## 6. Files and Code — the complete inventory

### Contribution 1 — Token optimization

| Path | What |
|---|---|
| `agent_kb/wiki/SCHEMA.md` | Compile contract |
| `agent_kb/README.md` | Layers, runbook, hard rules |
| `agent_kb/raw/STRATEGY_CREATION_GUIDE.md` | Immutable ground truth |
| `agent_kb/wiki/_tool/INDEX.md` | Navigator + documented seams |
| `agent_kb/wiki/_tool/topics/*.md` | 15 pages |
| `agent_kb/wiki/_tool/_token_report.md` | Before/after measurement |
| `scripts/compile_sdk_wiki.py` | The compiler |
| `quantjo_api/agent/system_prompt.py` | `_build_sdk_navigator()`, `SDK_NAVIGATOR` |
| `quantjo_api/agent/tools/read_sdk_page.py` | `read_sdk_page()` |
| `quantjo_api/agent/agents/code_expert.py` | `_sdk_reference`, tool registration |

### Contribution 2 — Regression harness

Entire `quantjo-agent-tests/` repository — see [04](04_regression_testing.md) §6 for the file-by-file
breakdown.

### Contribution 3 — Streaming

| Path | What |
|---|---|
| `quantjo_api/agent/agent_loop.py` | `_handle_stream_event()`, `TOOL_LABELS` |
| `quantjo_api/agent/event_emitter.py` | `emit_status` usage pattern |
| `quantjo-agent-tests/.../client/websocket_client.py` | Protocol consumer |

---

## 7. Design Decisions I Personally Made

| Decision | My reasoning | What I traded away |
|---|---|---|
| Navigator routing, not vector search | Small, closed, curated, cleanly-separated corpus → routing problem. Exact-ID can't retrieve the wrong chunk | Doesn't scale past a prompt-sized index |
| Immutable `raw/` + provenance frontmatter | LLM-generated docs must be auditable to ground truth | Compile-time discipline; duplication |
| Staged compile with human review | Catch a bad decomposition cheaply | Slower compile loop |
| Fall back to the full guide | Worst case should be cost, never correctness | Silent cost regression without alerting |
| Strip aliases from the runtime navigator | Big token win; the model needs triggers, not synonyms | Slightly weaker routing on odd phrasing |
| Three scoring layers, cheapest first | Short-circuit before paying for the judge | More machinery |
| Baseline diffing over absolute grading | Makes an uncalibrated judge genuinely useful | Can't certify correctness |
| Boundary pairs with negative assertions | Only way to catch over-correction | Requires maintaining paired probes |
| Three-state verdict (pass/fail/**flaky**) | Instability is information, not noise to collapse | Cost × k |
| Judge fails open | A flaky API must not block deploys | Silent loss of gating if unmonitored |
| Simulator blind to tool calls | Must match what a real user experiences | Can't assert tool behaviour from the user side |
| Status ephemeral, not messages | Transient UI state ≠ durable conversation state | Progress not recoverable after reload |
| Unknown tool → emit nothing | Never show raw identifiers to non-technical traders | New tools silently lack labels |

---

## 8. Production Improvements I'd Make to My Own Work

**Token optimization:** real tiktoken measurement · in-memory page cache · CI staleness gate on
`raw/` hash · alert on navigator fallback (a silent ~25× cost regression) · retrieval telemetry ·
A/B against the old path to prove quality held.

**Harness:** calibrate the judge against human labels (Cohen's κ) · measure judge variance to set the
regression threshold from data instead of intuition · parallelize case execution · wire the CI deploy
gate · populate `aggregate_tokens` from the `token_usage` WS event · enforce waiver expiry ·
version the judge prompt · trend dashboards.

**Streaming:** explicit `turn_complete` frame with `message_id` correlation · Redis pub/sub for
multi-instance fan-out · backpressure on slow consumers · structured logging instead of `print` ·
protocol versioning.

---

## 9. Common Interview Questions

1. What did you actually build?
2. What existed before you joined?
3. Which contribution are you proudest of and why?
4. What was the hardest technical problem?
5. Tell me about a bug you struggled with.
6. What did you measure, and what did you infer?
7. What would you do differently?
8. What's the weakest part of your work?
9. How did you know your changes worked?
10. How did you decide what to build?
11. What did you learn that surprised you?
12. Where did you get stuck, and what did you do?
13. How did you get feedback on your design?
14. What's a decision you'd reverse?
15. What did you build that nobody asked for?
16. How did your work change the team's process?
17. What would you tell the next intern?
18. What part of the codebase do you know best?
19. What part do you know least?
20. If you had three more months, what would you build?

---

## 10. Strong Answers

**Q3 — Proudest?**
> The boundary probe pairs in the test harness. Not because they're technically complex — Layer 3 is
> maybe eighty lines — but because they solve a problem I don't think conventional testing can. The
> characteristic failure of prompt engineering is over-correction: you fix one behaviour and the agent
> starts applying it everywhere. Standard per-case testing misses that completely, because the case
> you broke still passes its own assertions — it produces working code, just the wrong kind. Pairing
> the cases and requiring a negative assertion, that A must *not* exhibit B's pattern, is what makes
> the failure visible. That idea came from actually thinking about how this system fails rather than
> importing a testing pattern from ordinary software.

**Q6 — Measured vs inferred?**
> I want to be careful here because it's easy to overstate. Measured: prompt token counts, though with
> a `chars/4` heuristic rather than real tokenization, because tiktoken was blocked by our CodeArtifact
> pip auth. Also measured: judge scores per dimension across runs, and case durations. Inferred:
> latency improvement — I reasoned that removing nineteen thousand tokens of prefill reduces
> time-to-first-token, but I didn't benchmark it. Also inferred: everything in my scalability analysis,
> which is structural reasoning from the code rather than load-test data. And not measured at all:
> whether the navigator change affected output quality. The regression harness is the closest evidence
> I have, but I never ran a proper A/B against the old path, which is what would actually prove it.

**Q8 — Weakest part?**
> Verification of my own claims. Each of my three contributions has a measurement gap. The token
> numbers are a heuristic. The judge is uncalibrated against human labels, so it detects change well
> but can't certify correctness. And I never A/B tested the navigator against wholesale injection, so
> "quality held" is an inference from the harness rather than a result. The through-line is that I was
> better at building the thing than at proving it worked, and if I did the internship again I'd budget
> a third of the time for measurement instead of treating it as a follow-up.

**Q14 — A decision you'd reverse?**
> Deferring the CI staleness gate on the knowledge base. The wiki is compiled from a source guide, and
> nothing enforces that they stay in sync — it's a documented manual step. I knew that was a gap when
> I built it and told myself it was a follow-up. But it's the kind of gap that degrades silently: the
> guide gets edited, nobody recompiles, and the agent is now grounded in stale documentation with no
> signal that anything is wrong. It's a twenty-line CI job that hashes `raw/` and compares it to the
> stored compile state. I should have built it at the same time as the compiler, because the cost of
> doing it later is that in between you're accumulating invisible risk.

**Q17 — What would you tell the next intern?**
> Two things. First, read the git history before the code — the commits told me why the architecture
> was the way it was, including a whole multi-agent-to-single-agent simplification that made the
> current design make sense. Without that I'd have "improved" things back toward a design they'd
> deliberately abandoned. Second, when you're working on an AI system, build the measurement before
> the optimization. I did it the other way round on all three of my projects and every one of them has
> a caveat attached to its result because of it.

---

## 11. Follow-up Questions

- "You keep saying what existed before you. Why are you so careful about that?"
  *(Good answer: because if I claim the platform and you ask me how the OAuth flow works, I'm caught —
  and then you'd doubt the parts I did build. Precision protects the real claims.)*
- "Three projects in one internship — how did you scope them?"
- "Which of the three had the most impact on users?"
- "Who reviewed your designs, and what did they push back on?"
- "You built the harness in a separate repo. Was that your call?"
- "What did you not finish?"
- "How did you validate the navigator didn't hurt output quality?"

---

## 12. Senior Engineer Perspective

What a senior engineer is looking for in an intern, and where you land:

| Signal | Evidence |
|---|---|
| **Judgment over enthusiasm** | Choosing navigator routing over a vector DB, and knowing when that flips |
| **Failure-mode thinking** | Navigator falls back · judge fails open · unknown tools emit nothing |
| **Reframing hard problems** | "Is it correct" → "did it get worse" |
| **Debugging under uncertainty** | The `AsyncSession` identity-map race |
| **Intellectual honesty** | Volunteering the heuristic, the uncalibrated judge, the missing A/B |
| **Knowing the ceiling** | The navigator stops working past a prompt-sized index |
| **Original thinking** | Boundary probe pairs for over-correction |

The single strongest thing you can do in an interview is **name your own gaps before you're asked**.
Every one of your three contributions has a real limitation. A candidate who says "here's what I built,
here's what I measured, here's what I only inferred, and here's the thing I'd fix first" is
dramatically more credible than one who presents three clean successes and then crumbles on the first
probing follow-up.

---

## 13. Common Mistakes

- **Inflating ownership.** You didn't build the platform, the auth system, the FastAPI backend, the
  agent architecture, or the WebSocket transport. You built three things *inside* an existing system.
  That is a strong internship. Claiming more makes it weak.
- **Claiming technologies from a résumé bullet.** No LangGraph, no ChromaDB, no tree-sitter, no
  FastMCP, no GitHub API, no embeddings. See [index.md](index.md).
- **Quoting 20K → 350 without the caveat.** Two different artifacts, and a heuristic. Precision here
  makes you sound rigorous; looseness gets you caught.
- **Saying "we" for everything.** Interviewers cannot assess "we." Use "I" for what you did and "the
  team" or "it already existed" for what you didn't.
- **Leading with the product.** You're an engineer. Two sentences of context, then architecture.
- **Not having the three stories rehearsed.** Behavioural questions are guaranteed. §5 is your prep.

---

## 14. Cheat Sheet

**Three contributions, one line each:**
1. **Token optimization** — compiled a 20K-token SDK guide into 15 retrievable pages + a ~350-token
   navigator + `read_sdk_page` tool. **19,659 → 792 always-on (~96%)**, ~2,068 typical (~89%). Falls
   back to the full guide on any failure.
2. **Regression harness** — standalone repo, 20 multi-turn cases, LLM user simulator, 3 layers
   (AST · LLM judge · boundary pairs), majority vote with a `flaky` state, baseline diffing with a
   0.15 blocking threshold.
3. **Streaming** — `Runner.run_streamed` event dispatch → `TOOL_LABELS` → ephemeral `status` events
   over WebSocket. Fixed history pollution by moving from `emit_message` to `emit_status`.

**Three stories:** cost optimization (A) · the `AsyncSession` identity-map race (B) · making an
unverifiable system verifiable (C).

**Three honest caveats — volunteer these:**
- Token numbers are `chars/4`, not tiktoken (CodeArtifact blocked the install).
- The judge is uncalibrated against human labels → detects change, doesn't certify correctness.
- No A/B proving the navigator didn't cost quality; scalability claims are structural analysis, not
  load tests.

**The framing sentence:** *"The platform existed. I owned three pieces of the AI layer: making code
generation cheaper, making it testable, and making it visible."*
