# Agent Architecture

> **Note:** the orchestration framework is the **OpenAI Agents SDK** (`openai-agents`), not LangGraph.
> There is no graph, no node/edge definition, no state machine library. Don't claim LangGraph.

---

## 1. Interview Answer (2–3 minutes)

> The agent layer went through an architectural simplification while I was there, and the end state
> is worth explaining because the reasoning is the interesting part.
>
> It's a single-agent loop with a sub-agent as a tool. Every conversational turn starts and ends in
> the main agent — Jo. Jo handles conversation, asset resolution, and result analysis. Code generation
> is *not* a handoff to another agent; it's a tool call named `propose_code_change`, and internally
> that tool invokes a specialist called Codex in its own ephemeral session with its own model and its
> own tool set.
>
> The distinction matters. With a handoff, control transfers — the specialist owns the conversation
> and has to know how to hand back, and you get failure modes where it answers something outside its
> domain or gets stuck. With a tool call, control never leaves Jo. Codex gets a narrowly scoped brief,
> returns a changeset, and Jo continues the conversation. The routing logic collapses into "which tool
> do I call," which is what language models are already good at.
>
> Codex runs on a code-tuned model with medium reasoning effort, and it has four tools: read the
> current strategy file, read an SDK documentation page, resolve asset symbols, and propose files. It
> never talks to the user directly. Its output goes into an atomic changeset — written to a pending
> location in S3, surfaced as a diff, and only promoted to the live file when the trader accepts.
>
> The other piece I'd highlight is context management. The system prompt is assembled at module import
> from static blocks with a hard rule that nothing dynamic goes in it — that keeps it prompt-cacheable.
> Per-turn context, like which tab the user is looking at or feedback from a rejected changeset, goes
> in as ephemeral blocks through a session callback, and those never persist to conversation history.

---

## 2. What Problem Were We Solving?

**Business.** The agent has to do several different jobs — conversation, code generation, asset
lookup, result analysis, launching backtests — without confusing the trader or producing unwanted
side effects. A specific failure mattered: the agent generating a changeset when the user just wanted
an explanation. That's the "unwanted-changeset trap," and it was fixed by giving `focus_view` the
ability to show code without invoking Codex.

**Technical.**
- **Model specialization.** Conversation and code generation want different models and different
  reasoning budgets.
- **Context isolation.** Codex shouldn't inherit the whole chat history; it needs a focused brief.
- **Cost.** Different models and prompt sizes per job.
- **Determinism of control flow.** With multiple peer agents, you must reason about who is in control.

**Why the earlier design wasn't enough.** The original architecture used handoffs to specialist
agents (`code_expert`, `backtest_runner`, `optimization_planner`). Handoff systems require every
specialist to know when to hand back, prompts accumulate "hand back silently on out-of-scope asks"
rules, and control flow becomes hard to reason about. Collapsing to one conversational agent with
tools removed a whole category of bugs.

---

## 3. Architecture

```
                      user message (WebSocket)
                               │
        ┌──────────────────────▼───────────────────────────┐
        │ AgentLoop.process_message()                      │
        │  1. Guardrails.check_rate_limit(user_id)         │
        │  2. billing.check_token_quota(db, user_id)       │
        │  3. StateCoordinator.acquire_lock(message_id)    │
        │       duplicate → return cached response         │
        │       conflict  → error                          │
        └──────────────────────┬───────────────────────────┘
                               ▼
        ┌──────────────────────────────────────────────────┐
        │ _run_turn()                                      │
        │   ProjectStateSession (ai_state.conversation)    │
        │     wrapped by OpenAIResponsesCompactionSession  │
        │       model="gpt-5-nano", manual trigger         │
        │   build_ephemeral_blocks(view, pending_changeset,│
        │       rejection, is_new_project)                 │
        │   → RunConfig(session_input_callback=callback)   │
        └──────────────────────┬───────────────────────────┘
                               ▼
   ┌───────────────────────────────────────────────────────┐
   │ Runner.run_streamed(main_agent, input=user_message,   │
   │                     max_turns=10)                     │
   │                                                       │
   │   main_agent "Jo"                                     │
   │     PERSONA + CODE_GUARDRAILS + STRATEGY_INTAKE       │
   │     + tool behaviour blocks                           │
   │                                                       │
   │     tools:                                            │
   │       resolve_assets       → symbol candidates        │
   │       get_backtest_results → metrics + trade log      │
   │       invoke_action        → open/focus a view        │
   │       propose_code_change ─────────┐                  │
   └────────────────────────────────────┼──────────────────┘
                                        ▼
        ┌────────────────────────────────────────────────┐
        │ code_expert "Codex"  — TOOL, NOT HANDOFF       │
        │   model = gpt-5.3-codex                        │
        │   reasoning effort = medium                    │
        │   fresh ephemeral session per invocation:      │
        │     HEAD backtest.py + trader intent           │
        │     + optional rejection feedback              │
        │                                                │
        │   instructions = PERSONA + CODE_GUARDRAILS     │
        │     + tool behaviour + SDK NAVIGATOR (~350 tok)│
        │                                                │
        │   tools:                                       │
        │     read_head_file  → current accepted code    │
        │     read_sdk_page   → one KB topic page        │
        │     resolve_assets  → symbols                  │
        │     propose_files   → validate + write pending │
        └───────────────────┬────────────────────────────┘
                            ▼
              SDK validation → S3 pending/backtest.py
              → artifact_update event → trader accepts
              → promoted to head/backtest.py
```

---

## 4. Deep Technical Explanation

### Sub-agent as tool, not handoff

From the `agent_loop.py` docstring:

> Every chat turn starts and ends in `main_agent`. There are no specialist agents and no handoffs;
> code-edit work is a tool call (`propose_code_change`) that internally invokes the Codex sub-agent in
> its own ephemeral session.

| | Handoff | Sub-agent as tool |
|---|---|---|
| Control | transfers to specialist | never leaves Jo |
| Context | specialist inherits conversation | specialist gets a scoped brief |
| Return path | specialist must know to hand back | function returns |
| Failure mode | stuck in the wrong agent | tool returns an error Jo can handle |
| Prompt cost | hand-back rules in every specialist | none needed |

The `propose_code_change` docstring notes the practical consequence: *"Codex's system prompt
(CODE_GUARDRAILS + SDK_GUIDE) is fixed at the agent"* — Codex's context is constructed per call, not
inherited. That's the isolation you want. Jo's chatty history doesn't dilute a code-generation prompt.

### Model selection

| Agent | Model | Why |
|---|---|---|
| `main_agent` (Jo) | conversational model | Conversation, tool routing, explanation |
| `code_expert` (Codex) | `gpt-5.3-codex`, `Reasoning(effort="medium")` | Code-tuned; explicit reasoning budget |
| Compaction | `gpt-5-nano` | Summarization is easy; use the cheapest capable model |
| Judge (harness) | `gpt-4o-mini` | Evaluation at volume |
| User simulator (harness) | `gpt-4o` | Must convincingly play a human |

**Matching model to task is a cost/quality lever, and being able to explain each choice is a strong
signal.** Using a frontier model for compaction would be pure waste; using nano for code generation
would be malpractice.

### Prompt composition — static and cacheable

`system_prompt.py` exports named string constants composed at import:

```python
_CODE_EXPERT_SYSTEM_PROMPT = "\n\n".join([
    PERSONA, CODE_GUARDRAILS, "## Tool Behaviour",
    TOOL_BEHAVIOUR_RESOLVE_ASSETS, TOOL_BEHAVIOUR_READ_HEAD_FILE,
    TOOL_BEHAVIOUR_PROPOSE_FILES, _TOOL_BEHAVIOUR_READ_SDK_PAGE,
    _sdk_reference,
])
```

The module docstring states the rule: **no dynamic content in the system prompt.** No project name, no
file content, no user data. That's what keeps the prefix byte-identical across requests and therefore
prompt-cacheable. The changelog records "Optimized prompts for caching — static content first, dynamic
content last (50-90% token cost reduction)."

### Ephemeral context — the other half

Dynamic per-turn context goes through a separate path:

```python
ephemeral_blocks = build_ephemeral_blocks(
    view=view,                                     # which tab the user is looking at
    pending_changeset=project.current_changeset_id is not None,
    rejection=rejected_changeset,                  # feedback from a rejected proposal
    is_new_project=head_version is None,           # triggers STRATEGY_INTAKE
)
callback = make_session_input_callback(ephemeral_blocks)
Runner.run_streamed(..., run_config=RunConfig(session_input_callback=callback))
```

These are injected into the turn's input by a callback and **never persisted** to conversation
history. *Why?* Because "the user is looking at the backtest results tab" is true now and false in
three turns. Persisting it would poison future turns with stale state and cost tokens forever.

Note `is_new_project` — it injects a marker that activates the `STRATEGY_INTAKE` block, which stops
the agent from generating code on a trader's first message and makes it act as a copilot instead:
read the idea back, name what's ambiguous, recommend a minimum viable version, ask *one* question. A
soft behavioural rule, conditionally activated by ephemeral state. Elegant, and worth mentioning.

### Conversation persistence and compaction

`ProjectStateSession` persists conversation into `Project.ai_state.conversation` (JSONB). It's wrapped
by `OpenAIResponsesCompactionSession` with `should_trigger_compaction=lambda _: False` — **manual
control**, deliberately.

```python
COMPACTION_THRESHOLD = 20  # non-user items
async def _maybe_compact(session, conversation):
    non_user_items = [i for i in conversation
                      if not (i.get("role") == "user" or i.get("type") == "compaction")]
    if len(non_user_items) >= COMPACTION_THRESHOLD:
        await session.run_compaction({"force": True})
```

Two design points:

- **Counting non-user items only.** User messages are the signal you least want to lose; tool calls
  and assistant messages are what bloat context.
- **Compaction runs in a background task after the response.** `asyncio.create_task(_maybe_compact(...))`
  means the user never waits for summarization.

Historical note worth knowing: the changelog records switching to `OpenAIResponsesCompactionSession`
because the previous approach corrupted reasoning-message pairs. Reasoning items must stay paired with
their outputs; naive truncation breaks that and the API rejects the conversation.

### Idempotency and locking

```python
success, error = await StateCoordinator.acquire_lock(db, project, message_id)
if error == "duplicate":
    cached_response = project.ai_state.get("last_response", {})
    return {**cached_response, "cached": True}
```

An optimistic lock keyed on `message_id` with the last response cached in `ai_state`. A retry with the
same `message_id` returns the cached response instead of re-running the agent.

**Why this matters specifically here:** WebSockets drop, clients reconnect and retry. Without
idempotency a network blip means the agent runs twice — double the tokens, double the cost, and
possibly two competing changesets. This is the LLM-era version of double-charging a payment.

### Guardrails and quotas

- `Guardrails.check_rate_limit(user_id)` — per-user request throttling.
- `billing.check_token_quota(db, user_id)` — monthly plan-based token budget, raising `QuotaExceeded`
  with `meter`, `used`, `limit`, `resets_at`, `plan`.
- `max_turns=10` on the runner — bounds the tool-calling loop so a confused agent can't spin forever.

That last one is easy to overlook and important: an agent that keeps calling tools without converging
is an unbounded cost.

### Token accounting

```python
wrapper = getattr(result, "context_wrapper", None)
usage = getattr(wrapper, "usage", None)
```

`RunResultStreaming` has no `.usage`. Usage comes from `context_wrapper`, which accumulates across
tool calls **including the nested Codex sub-run** — so one `TokenUsage` row captures the whole turn
including the sub-agent. Cached tokens are pulled from `usage.input_tokens_details.cached_tokens`,
which is how prompt-cache effectiveness is measured.

---

## 5. My Actual Contribution

**What existed** — the agent architecture: the loop, the main agent, the sub-agent-as-tool pattern,
sessions, compaction, guardrails, the changeset system, `StateCoordinator`.

**What I built** — within that architecture: the SDK navigator that replaced wholesale guide
injection in Codex's prompt, the `read_sdk_page` tool and its registration in Codex's tool list, the
`_TOOL_BEHAVIOUR_READ_SDK_PAGE` instruction block, the streaming event dispatch and `TOOL_LABELS`,
and the regression harness that validates the whole thing.

**What changed** — Codex's prompt shrank ~96%, its retrieval behaviour became observable, and agent
changes became testable.

**Result** — cheaper code generation with a quality gate behind it.

---

## 6. Files and Code

| Path | Contents |
|---|---|
| `agent/agent_loop.py` | `AgentLoop.process_message`, `_run_turn`, `_handle_stream_event`, `TOOL_LABELS`, `COMPACTION_THRESHOLD`, `_maybe_compact`, `_extract_response` |
| `agent/agents/main_agent.py` | Jo — persona, tools, model |
| `agent/agents/code_expert.py` | Codex — `gpt-5.3-codex`, `Reasoning(effort="medium")`, prompt composition, tool list |
| `agent/system_prompt.py` | `PERSONA`, `CODE_GUARDRAILS`, `STRATEGY_INTAKE`, `TOOL_BEHAVIOUR_*`, `SDK_GUIDE`, `SDK_NAVIGATOR` |
| `agent/tools/` | `resolve_assets.py`, `get_backtest_results.py`, `propose_code_change.py`, `propose_files.py`, `read_head_file.py`, `read_sdk_page.py` |
| `agent/session.py` | `ProjectStateSession` |
| `agent/ephemeral.py` | `build_ephemeral_blocks`, `make_session_input_callback` |
| `agent/context.py` | `AgentContext` — db, emitter, user_id, project_id, cancel_event |
| `agent/services/state_coordinator.py` | Optimistic lock, idempotency, state updates |
| `agent/services/guardrails.py` | `Guardrails`, `GuardrailError` |
| `agent/services/changeset_handler.py` | Accept/reject, pending→head promotion |
| `agent/action_registry.py`, `actions.py` | Chip definitions and contextual gating |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| Sub-agent as tool, not handoff | Control never leaves Jo; no hand-back logic | Sub-agent can't drive multi-turn dialogue |
| Fresh ephemeral session for Codex | Focused context; no chat dilution | Loses conversational nuance not in the brief |
| Static system prompt | Prompt-cacheable | Nothing personalized in the prefix |
| Ephemeral blocks via callback | Fresh per-turn context, no history pollution | Extra indirection |
| Manual compaction trigger | Explicit control over when and what | Must maintain the threshold logic |
| Compaction on non-user items | Preserve user intent, compress the rest | Assumes user turns are short |
| Background compaction | User doesn't wait | Race with the next turn |
| Idempotency on `message_id` | Reconnects don't double-charge | Cached response may be stale on legitimate retry |
| `max_turns=10` | Bounds runaway tool loops | A genuinely complex task could truncate |
| Atomic changeset | AI edits reviewable | Extra UX step |
| Model per task | Cost/quality matched | More models to track and version |

---

## 8. Production Improvements

1. **Tracing per turn.** The Agents SDK supports tracing; a UI over it would make debugging a bad turn
   dramatically faster than reading logs.
2. **Prompt versioning + A/B.** Ship prompt changes to a traffic slice, compare judge scores.
3. **Semantic caching** on near-identical requests.
4. **Streaming Codex's output** so the diff renders progressively.
5. **Tool-call budgets per turn**, not just `max_turns` — a cost ceiling per turn.
6. **Fallback model on provider outage** — the agent path is a hard dependency on one vendor today.
7. **Structured tool errors** so Jo can distinguish "asset not found" from "provider down" and say
   something useful.
8. **Persist ephemeral decisions for audit** — you can't currently reconstruct exactly what context a
   past turn saw, which makes debugging historical bad output hard.

---

## 9. Common Interview Questions

1. Walk me through the agent architecture.
2. Why a sub-agent as a tool instead of a handoff?
3. What changed from the earlier multi-agent design and why?
4. What tools does each agent have?
5. Why different models per agent?
6. How is the system prompt composed and why does that matter?
7. What's ephemeral context and why isn't it persisted?
8. How does conversation history not grow unbounded?
9. Why compact on non-user items only?
10. Why is compaction manual rather than automatic?
11. How do you prevent duplicate processing?
12. What's `max_turns` for?
13. How does an atomic changeset work?
14. What happens when a user rejects generated code?
15. How does the agent know which asset the user means?
16. How is token usage tracked across a nested sub-agent run?
17. What's the "unwanted changeset trap"?
18. How does `STRATEGY_INTAKE` work and when is it active?
19. What happens if OpenAI is down mid-turn?
20. How do you know a prompt change didn't break anything?

---

## 10. Strong Answers

**Q2 — Sub-agent as tool vs handoff.**
> With a handoff, control transfers — the specialist owns the conversation and has to know when and
> how to give it back. In practice that means every specialist prompt accumulates rules like "hand
> back silently on out-of-scope asks," and you get failure modes where the code specialist starts
> answering questions about backtest results because the user asked while it had control. With a tool
> call, control never leaves the main agent. Codex receives a scoped brief, does one job, and returns
> a changeset — it's a function call. Routing collapses to "which tool do I call," which is exactly
> what these models are trained to do well. We'd previously had three specialist agents with handoffs
> and collapsed to this, and it removed a whole class of control-flow bugs.

**Q7 — Ephemeral context.**
> It's per-turn state that matters now and not later — which tab the user is looking at, whether
> there's a pending changeset, feedback from a rejected proposal, whether this is a brand-new project.
> It's built by `build_ephemeral_blocks` and injected into the turn's input through a session
> callback, and critically it never gets written into conversation history. Two reasons. It goes stale
> — "the user is viewing backtest results" is wrong three turns later, and a model reading it as
> history will act on stale state. And it would cost tokens forever, since every persisted item is
> re-sent on every subsequent turn. The mirror-image rule is that the system prompt holds only static
> content, so it stays prompt-cacheable. Static in the prefix, dynamic in the ephemeral blocks, and
> nothing dynamic persists.

**Q11 — Duplicate processing.**
> Optimistic locking on `message_id`. Before running, `StateCoordinator.acquire_lock` checks whether
> that `message_id` was already processed; if so, it returns the response cached in
> `project.ai_state.last_response` with a `cached: true` flag instead of re-running. This matters more
> than it would in a normal API because the transport is a WebSocket — connections drop, clients
> reconnect and retry, and without idempotency a network blip costs the user a second full agent turn
> in tokens and could produce two competing changesets. It's the same reasoning as an idempotency key
> on a payments API.

**Q19 — OpenAI down mid-turn.**
> Today it degrades but doesn't crash. `_run_turn` wraps the whole run in a try/except that logs,
> emits an `error` frame over the WebSocket, and returns a user-facing message asking them to try
> again. The turn's state is cleaned up because the response is still persisted through
> `_persist_response_simple`. What's missing is anything smarter: no retry with backoff, no fallback
> model, no circuit breaker. So a provider outage is a full outage of the core product feature. If I
> were hardening it I'd add retry with jitter for transient errors, a circuit breaker so we fail fast
> rather than making every user wait for a timeout, and ideally a secondary provider for the
> conversational path — code generation is harder to substitute because it's tuned to a specific model.

---

## 11. Follow-up Questions

- "Compaction runs in a background task. What if the next turn starts before it finishes?"
- "The compaction model is `gpt-5-nano`. What happens if it summarizes away something critical?"
- "Codex gets a fresh session each call. How does it handle 'make it like we discussed earlier'?"
- "`max_turns=10` — what happens on turn 11?"
- "How do you debug a turn that produced bad output three days ago?"
- "Two browser tabs, two changesets. What happens?"
- "Reasoning-message pair corruption — what was that bug exactly?"

---

## 12. Senior Engineer Perspective

**The handoff→tool simplification is the headline.** It's an architecture that was made *smaller*, and
senior engineers value deletion. Be able to name the specific bug class it eliminated.

**Context engineering discipline** — static/cacheable vs ephemeral/non-persisted — is the highest-value
concept in this file. It's the same instinct as separating configuration from state.

**Cost is a first-class concern** here in a way it isn't in most systems: model selection, prompt
caching, compaction, `max_turns`, quotas, and per-turn token accounting are all cost controls. Framing
them as a coherent cost architecture rather than a list of features reads as senior.

**They'll probe reliability.** Single-provider dependency, no retry, no circuit breaker, background
compaction races. Know these.

---

## 13. Common Mistakes

- **Claiming LangGraph.** It's the OpenAI Agents SDK. There is no graph.
- **Describing it as multi-agent.** It's *single*-agent with a sub-agent as a tool. That precision is
  the whole interesting part.
- **Not knowing why the system prompt is static.** Prompt caching. It's the reason for the rule.
- **Forgetting idempotency.** It's a subtle, production-grade concern that shows maturity.
- **Saying "the agent decides what to do" vaguely.** Name the tools, name the routing mechanism.

---

## 14. Cheat Sheet

- **Framework:** OpenAI Agents SDK (`openai-agents`). `Agent`, `Runner.run_streamed`, `@function_tool`,
  `RunConfig`. **Not LangGraph.**
- **Shape:** single agent (Jo) + **sub-agent as tool** (Codex via `propose_code_change`). No handoffs.
- **Jo's tools:** `resolve_assets`, `get_backtest_results`, `invoke_action`, `propose_code_change`.
- **Codex:** `gpt-5.3-codex`, `Reasoning(effort="medium")`, fresh ephemeral session.
  Tools: `read_head_file`, `read_sdk_page`, `resolve_assets`, `propose_files`.
- **Pre-flight:** rate limit → token quota → optimistic lock (`message_id` → cached response).
- **`max_turns=10`** bounds the tool loop.
- **Prompt:** static blocks joined at import ⇒ cacheable. **No dynamic content in the system prompt.**
- **Ephemeral:** `build_ephemeral_blocks(view, pending_changeset, rejection, is_new_project)` →
  `session_input_callback`. Never persisted.
- **Session:** `ProjectStateSession` (JSONB `ai_state.conversation`) wrapped by
  `OpenAIResponsesCompactionSession` (`gpt-5-nano`, manual trigger, threshold **20 non-user items**,
  run in a background task).
- **Usage:** `result.context_wrapper.usage` (not `RunResultStreaming.usage`); includes the Codex
  sub-run; `input_tokens_details.cached_tokens` measures cache hits.
- **Changeset:** `propose_files` → SDK validation → S3 `pending/` → `artifact_update` → accept →
  `head/`.
