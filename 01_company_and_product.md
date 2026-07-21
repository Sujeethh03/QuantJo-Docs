# Company and Product

## 1. Interview Answer (2–3 minutes)

> I interned at Quantjo, which builds an AI platform for algorithmic trading on the Indian markets —
> NSE and NFO. The core problem it solves is that there are a lot of experienced traders who
> understand markets deeply — indicators, entries, risk — but can't write Python. Historically, if
> you wanted to backtest an idea, you either learned to code or you paid someone. Quantjo lets you
> describe the strategy in plain English to an AI assistant called Jo, and Jo writes the actual
> strategy code, runs the backtest, and shows you the results.
>
> Architecturally it's two services. A FastAPI backend — about 54 REST endpoints plus one WebSocket
> endpoint — with PostgreSQL for metadata and S3 for the actual strategy files and backtest
> artifacts. And a Next.js frontend where the trader chats on the left and sees a code editor,
> backtest charts, and results tabs on the right.
>
> The AI layer is the interesting part. It's built on the OpenAI Agents SDK. There's a main agent,
> Jo, that handles conversation, and code generation is delegated to a specialist sub-agent we called
> Codex, which runs on a code-tuned model with its own tool set and its own ephemeral context. When
> Codex proposes code, it goes into the project as a pending changeset — the trader sees a diff and
> explicitly accepts or rejects it before it becomes the live strategy. So the AI never silently
> mutates the user's work.
>
> My team owned that AI layer specifically — the agent, its prompts, its tools, and the quality of
> what it produced. I worked on three things: cutting the token cost of the code-generation path,
> building the regression test harness that told us whether prompt changes broke anything, and the
> real-time streaming that shows the trader what the agent is doing while it's doing it.

---

## 2. What Problem Were We Solving?

**Business problem.** Retail and semi-professional traders in India have strategy ideas but not
Python skills. The tooling gap is the bottleneck between having an idea and knowing whether it makes
money. Every day of that gap is a day they're trading on intuition instead of evidence.

**Technical problem.** Natural language → correct, executable, SDK-conformant Python is a hard
generation problem. The code has to:
- subclass `Strategy` from the in-house `quantjo-sdk`
- declare an `assets` dict with named roles (`primary`, secondary)
- implement `generate_signals()` returning `{asset_name: DataFrame}`
- use `ExitRule` for fixed-percentage stops but inline `df['exit_*']` columns for path-dependent exits
- be written in Jupytext percent format with markdown cells a non-coder can read

Any one of those wrong and the backtest either crashes or — worse — silently produces wrong numbers
a trader might act on financially.

**Why existing approaches weren't enough.** A code-completion tool like Copilot has no knowledge of
a private SDK. A generic ChatGPT wrapper hallucinates SDK APIs that don't exist. A template library
handles the five strategies you anticipated and nothing else. The platform needed grounded generation
against a proprietary SDK, plus a human accept/reject gate, plus a way to *know* when the AI got worse.

---

## 3. Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Next.js 16 frontend (quantjo-ui-saas)                         │
│  chat pane  │  Monaco editor  │  backtest tabs  │  dashboard   │
└──────────────────────┬─────────────────────────────────────────┘
                       │  WebSocket  /ws/projects/{id}
                       │  + ~54 REST endpoints (auth, projects, backtests)
┌──────────────────────▼─────────────────────────────────────────┐
│  FastAPI backend (quantjo-saas / quantjo_api)                  │
│                                                                │
│   routes/websocket.py ──► AgentLoop.process_message()          │
│                              │                                 │
│                              ├─ Guardrails (rate limit)        │
│                              ├─ billing.check_token_quota()    │
│                              ├─ StateCoordinator (optimistic   │
│                              │    lock + idempotency)          │
│                              └─ Runner.run_streamed(main_agent)│
│                                     │                          │
│                                     ├─ resolve_assets          │
│                                     ├─ get_backtest_results    │
│                                     ├─ invoke_action           │
│                                     └─ propose_code_change ────┼──► code_expert
│                                                                │    (gpt-5.3-codex)
│   services/  models/  schemas/  core/                          │      ├ read_head_file
└──────────┬──────────────────────────┬──────────────────────────┘      ├ read_sdk_page
           │                          │                                 └ propose_files
    PostgreSQL 17               AWS S3 (ap-south-1)
    users, projects,            {user_id}/projects/{project_id}/
    chat_messages,                head/backtest.py
    backtests, token_usage        pending/backtest.py
                                  backtests/{id}/*.csv|.json|.html
```

**Request flow for "build me an RSI strategy":**

1. Frontend sends `{"type":"user_message","content":"...","message_id":uuid}` over the open WebSocket.
2. `routes/websocket.py` spawns `_process_and_emit()` as a background task with its **own**
   `AsyncSession` (per-turn isolation — see file 08 for why this matters).
3. `AgentLoop.process_message()` runs pre-flight: rate limit → token quota → optimistic lock.
4. `Runner.run_streamed(main_agent, ...)` starts. Events stream back as they happen.
5. Jo decides this needs code, calls `propose_code_change` — which internally invokes `code_expert`.
6. Codex reads the SDK navigator, calls `read_sdk_page("template-structure")`, writes code, calls
   `propose_files`.
7. Code is validated against the SDK, written to S3 `pending/`, and an `artifact_update` event fires.
8. Trader clicks Accept → `pending/backtest.py` is promoted to `head/backtest.py`.

---

## 4. Deep Technical Explanation

**Why two projects, not one.** The backend depends *only* on `quantjo-sdk` — never on platform
internals. That boundary means the SaaS can be deployed, versioned, and tested independently of the
backtesting engine, and the SDK can be mocked in tests.

**Why atomic changesets.** An AI that edits your file in place is terrifying when the file is a
trading strategy. The changeset model — write to `pending/`, show a diff, promote to `head/` only on
explicit accept — makes every AI edit reversible and reviewable. It's the same insight as a pull
request. Rejection isn't a dead end either: rejecting with feedback re-invokes the agent with the
rejection context so it can try again.

**Why S3 path convention instead of a database table.** There used to be a `project_artifacts` table.
It was dropped. Files now live at a deterministic path derived from `Project.s3_prefix` =
`{user_id}/projects/{project_id}`. Fewer moving parts, no table/storage divergence, no orphaned rows.
The tradeoff is you lose cheap SQL queries over file metadata — acceptable when the file set per
project is tiny and fixed.

**Why UUID primary keys.** Sequential integer IDs let an attacker enumerate `/projects/1`,
`/projects/2`. UUIDs make the ID space unguessable, which is a meaningful defense for a product where
a project *is* someone's proprietary trading edge.

**Why per-user data providers.** Market data comes from Zerodha (OAuth), Databento, and Massive (API
keys). Rather than one shared platform account, each user connects their own credentials, stored
Fernet-encrypted (AES-128-CBC + HMAC-SHA256). This is both a licensing requirement — market data
vendors bill per-seat — and a blast-radius reduction.

---

## 5. My Actual Contribution

**What existed** — the whole platform. FastAPI service, PostgreSQL schema, S3 integration, JWT auth,
the agent architecture, the changeset system, the WebSocket transport, the Next.js frontend.

**What I built** — three things inside the AI layer:
1. The compiled knowledge base and navigator that replaced wholesale prompt injection.
2. `quantjo-agent-tests` — a standalone regression harness (its own repo, own package, own CLI).
3. The tool-progress streaming path that surfaces agent activity to the UI in real time.

**What changed** — the code-generation prompt went from ~20K always-on tokens to ~350; we gained the
ability to detect prompt regressions before shipping; and the UI stopped being a black box during
the 30–60 seconds an agent turn takes.

**Result** — ~96% reduction in always-on prompt tokens on the code path, a measurable quality gate on
agent changes, and a visibly more responsive product.

> **Honesty note for interviews:** on this file specifically, most of what you're describing is
> context, not contribution. Be explicit about that. "The platform existed; I owned three pieces of
> the AI layer" is the right framing, and it makes your actual claims more credible, not less.

---

## 6. Files and Code

| Area | Path |
|---|---|
| FastAPI app entry | `quantjo-saas/quantjo_api/main.py` |
| Config (Pydantic settings) | `quantjo_api/config.py` |
| DB engine | `quantjo_api/core/database.py` |
| JWT / hashing | `quantjo_api/core/security.py` |
| ORM models | `quantjo_api/models/` — `user.py`, `project.py`, `backtest.py`, `chat_message.py`, `token_usage.py`, `data_provider.py` |
| Routes | `quantjo_api/routes/` — `auth.py`, `projects.py`, `websocket.py`, `backtests.py`, `providers.py`, `assets.py` |
| Agent layer | `quantjo_api/agent/` — `agent_loop.py`, `system_prompt.py`, `event_emitter.py`, `connection_manager.py`, `session.py`, `ephemeral.py`, `context.py` |
| Agents | `quantjo_api/agent/agents/` — `main_agent.py`, `code_expert.py` |
| Agent tools | `quantjo_api/agent/tools/` — `read_sdk_page.py`, `read_head_file.py`, `propose_files.py`, `propose_code_change.py`, `resolve_assets.py` |
| Knowledge base | `quantjo-saas/agent_kb/` |
| Test harness | `quantjo-agent-tests/src/quantjo_agent_tests/` |
| Frontend | `quantjo-ui-saas/src/app/`, `src/components/features/`, `src/actions/` |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| SDK-only backend dependency | Clean boundary, independent deploy, mockable | Every SDK bump is a coordinated upgrade |
| Atomic changesets | AI edits are reviewable and reversible | Extra UX step; extra state (`current_changeset_id`) |
| S3 path convention, no artifact table | One source of truth | No SQL queries over file metadata |
| UUID PKs | Anti-enumeration | 16 bytes vs 4; index bloat at scale |
| Per-user provider credentials | Licensing + blast radius | Onboarding friction; per-user token refresh |
| WebSocket for chat | Server-push progress during long turns | Stateful connections complicate horizontal scaling |
| Postgres + S3 split | Metadata queryable, blobs cheap | Two systems to keep consistent |

---

## 8. Production Improvements

- **Async backtest execution.** Backtests currently run synchronously and block. FAANG-grade: enqueue
  to SQS, execute on AWS Batch or ECS, notify completion over the existing WebSocket.
- **Read replicas** for the project/backtest list endpoints; the write path is tiny by comparison.
- **CDN-fronted result artifacts** instead of minting presigned URLs per request.
- **Multi-region** for latency — the platform serves Indian markets from `ap-south-1`, which is
  already right, but DR is single-region today.
- **Feature flags** on agent behavior so prompt changes ship to 1% of traffic first.
- **Cost attribution per user per turn** — `token_usage` captures the data; nothing surfaces it as a
  per-tenant unit-economics dashboard.

---

## 9. Common Interview Questions

1. What does the product do?
2. Who is the user, and why can't they just write Python?
3. Walk me through the architecture.
4. Why FastAPI and not Django or Flask?
5. Why PostgreSQL and S3 rather than one of them?
6. What's an atomic changeset and why does it exist?
7. Why UUID primary keys?
8. What happens when a user rejects generated code?
9. How does the backend know which market data provider to use?
10. Why are provider credentials per-user?
11. How is the SDK boundary enforced?
12. What's stored in Postgres vs S3?
13. How does a project get from empty to having a strategy?
14. What was your team responsible for?
15. What was *your* responsibility inside that?
16. How many endpoints, and how are they organized?
17. What's the role of the WebSocket vs the REST API?
18. How do you handle a user sending two messages at once?
19. What's the biggest technical risk in this product?
20. If you had another three months, what would you build?

---

## 10. Strong Answers

**Q3 — Walk me through the architecture.**
> Two services. Next.js frontend, FastAPI backend. The backend has a conventional layered structure —
> routes, services, models, schemas — plus an `agent/` package that's the AI layer. Postgres holds
> users, projects, chat history, backtest metadata, and token usage. S3 holds the actual strategy
> files and backtest output at a deterministic prefix per project. The chat runs over a WebSocket
> because an agent turn takes 30 to 60 seconds and we need to push progress; everything else is REST.

**Q6 — What's an atomic changeset?**
> When the agent writes code, it doesn't touch the live file. It writes to `pending/backtest.py` in
> S3 and sets `current_changeset_id` on the project. The frontend gets an `artifact_update` event and
> renders a diff. Only when the trader clicks Accept does `pending` get promoted to `head`. It's a
> pull request, essentially. The reason it matters here specifically is that the artifact is a
> trading strategy — if the AI silently degraded someone's stop-loss logic and they traded it, that's
> real money. The accept gate makes every AI edit a deliberate human decision.

**Q18 — Two messages at once?**
> The `ConnectionManager` keeps a single-slot buffer per connection with latest-wins semantics. If a
> turn is in flight and another message arrives, it replaces whatever was pending and we emit a
> `message_queued` event so the UI can show it. Separately there's an optimistic lock in
> `StateCoordinator` keyed on `message_id` — a duplicate `message_id` returns the cached response
> instead of re-running the agent, which makes retries idempotent. That matters because the client
> reconnects on network blips and would otherwise double-charge the user tokens.

**Q15 — Your responsibility.**
> Three things, all inside the AI layer. I cut the code-generation prompt from about 20K tokens to
> about 350 by compiling our SDK guide into a navigable wiki and giving the agent a tool to fetch
> pages on demand. I built the regression harness that runs 20 multi-turn conversations against a
> live server and scores the generated code with both AST assertions and an LLM judge. And I built
> the streaming path that surfaces which tool the agent is running to the UI in real time.

---

## 11. Follow-up Questions

- "You said the backend depends only on the SDK — what happens when the SDK makes a breaking change?"
- "Presigned URLs expire in an hour. What breaks, and how did you find out?"
- "If the WebSocket drops mid-turn, does the user lose the response?"
- "Where's the transactional boundary in an agent turn?"
- "You dropped the `project_artifacts` table. What did that cost you?"
- "How would you migrate a user's projects to a different region?"
- "What's your P99 latency on a code-generation turn and where does it go?"

---

## 12. Senior Engineer Perspective

A senior engineer will not be impressed by the feature list. They'll probe:

- **Consistency boundaries.** Postgres and S3 aren't in the same transaction. What happens if the S3
  write succeeds and the DB commit fails? (Honest answer: you get an orphaned pending file; the path
  convention means it's overwritten on the next proposal rather than leaking indefinitely.)
- **Failure modes under partial availability.** OpenAI is down. What does the user see?
- **Unit economics.** Every agent turn costs real money. Who pays, how is it metered, and what stops
  a user from burning $500 of tokens? (`billing.check_token_quota`, `Guardrails.check_rate_limit`.)
- **Blast radius of a bad prompt.** A one-line prompt edit can degrade every user's output silently.
  That's precisely why the regression harness exists — and it's the most senior-flavored thing in
  your whole internship. Lead with it.

---

## 13. Common Mistakes

- **Claiming the whole platform.** You didn't build the auth system. Saying "I built an AI trading
  platform" invites questions you can't answer and destroys trust when you can't.
- **Vagueness about the AI layer.** "It uses AI to generate code" is a non-answer. Name the SDK, the
  two agents, the tool list.
- **Over-indexing on the product pitch.** You're being interviewed as an engineer. Two sentences of
  product, then get to architecture.
- **Not knowing why, only what.** "We used Postgres and S3" — fine. "We used Postgres for queryable
  metadata and S3 for blobs because backtest HTML output is megabytes and doesn't belong in a row" —
  that's the answer.
- **Mentioning technologies from a résumé bullet you can't defend.** See the reality check in
  [index.md](index.md).

---

## 14. Cheat Sheet

- **Product:** plain-English → algorithmic trading strategy → backtest. NSE/NFO, Indian markets.
- **Stack:** FastAPI + Python 3.12 + SQLAlchemy 2.0 async + asyncpg + Pydantic v2 + PostgreSQL 17 +
  S3 (ap-south-1). Frontend Next.js 16 / React 19 / TypeScript / Tailwind / shadcn / Monaco.
- **AI:** OpenAI Agents SDK. `main_agent` (Jo) + `code_expert` (Codex, `gpt-5.3-codex`).
  `propose_code_change` is a tool, not a handoff.
- **Surface:** ~54 REST endpoints + `WS /ws/projects/{id}`.
- **Data:** Postgres = users, projects, chat_messages, backtests, optimizations, paper_trades,
  token_usage, data_providers. S3 = `{user_id}/projects/{project_id}/{head|pending}/backtest.py`.
- **Key pattern:** atomic changeset — pending → accept → head.
- **Auth:** JWT, 24h access / 30d refresh, cookie-based on the frontend.
- **My three:** navigator token optimization · regression harness · streaming.
