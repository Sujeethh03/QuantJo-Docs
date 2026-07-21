# System Design — End to End

---

## 1. Interview Answer (2–3 minutes)

> Let me trace one request all the way through, because that explains the architecture better than a
> component list.
>
> A trader types "build me an RSI mean reversion strategy" into a Next.js frontend. That goes over an
> already-open WebSocket to FastAPI — not REST, because an agent turn takes thirty to sixty seconds
> and we need to push progress back.
>
> The WebSocket route spawns the turn as a background asyncio task with its own database session, then
> runs pre-flight checks: per-user rate limit, monthly token quota, and an optimistic lock keyed on
> message ID so a reconnect-retry returns a cached response instead of running the agent twice.
>
> Then the agent runs. Conversation history comes from a JSONB column, wrapped in a compaction session
> that summarizes once there are twenty non-user items. Per-turn context — which tab they're on,
> whether there's a pending changeset — is injected via a callback and never persisted. The main agent
> decides this needs code and calls `propose_code_change`, which invokes a code specialist in its own
> ephemeral session. That specialist has a 350-token navigator listing fifteen SDK documentation
> topics; it fetches the one or two it needs by exact ID, then writes the strategy.
>
> Throughout, tool-call events stream back as status updates so the UI shows "Reading SDK reference",
> "Generating code" rather than a spinner.
>
> The generated file is validated against the SDK, written to S3 under a pending path, and surfaced as
> a diff. The trader accepts, and pending gets promoted to head. Later they run a backtest against
> that file; results land in S3 and summary metrics in Postgres so the dashboard can query them
> without touching object storage.
>
> Storage is split deliberately: Postgres for queryable metadata, S3 for blobs — strategy files,
> trade CSVs, and multi-megabyte interactive HTML reports.

---

## 2. What Problem Were We Solving?

**Business.** A multi-tenant SaaS where each user has private strategy code, private credentials, and
metered AI usage — with a core interaction that takes a minute and costs real money per invocation.

**Technical.**
- Long-running, expensive, stateful AI conversations
- Bidirectional real-time communication
- Large binary artifacts alongside queryable metadata
- Per-user isolation on every path
- Cost metering and quota enforcement
- Third-party dependencies: OpenAI, three market-data vendors, AWS

**Why a simpler design wasn't enough.** A plain CRUD app with a `POST /generate` endpoint fails on
every axis: no progress visibility, no cancellation, no idempotency across reconnects, no metering,
and a request timeout problem the moment generation exceeds a load balancer's idle limit.

---

## 3. Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│  BROWSER — Next.js 16 / React 19 / TS / Tailwind / shadcn / Monaco│
│                                                                   │
│  chat pane │ code editor │ backtest tabs │ project dashboard      │
│  JWT in cookies · server actions in src/actions/                  │
└───────────┬───────────────────────────────────┬───────────────────┘
            │ WebSocket (conversation)          │ REST (~54 endpoints)
            │ /ws/projects/{id}                 │ auth, projects, backtests,
            │                                   │ providers, templates
┌───────────▼───────────────────────────────────▼───────────────────┐
│  FASTAPI  (uvicorn, Python 3.12, fully async)                     │
│                                                                   │
│  routes/ ──► services/ ──► models/ ──► asyncpg                    │
│                                                                   │
│  agent/                                                           │
│    ConnectionManager (in-process: project_id → {connections})     │
│    AgentLoop                                                      │
│      ├ Guardrails.check_rate_limit                                │
│      ├ billing.check_token_quota                                  │
│      ├ StateCoordinator.acquire_lock (idempotency)                │
│      └ Runner.run_streamed(main_agent, max_turns=10)              │
│           ├ EventEmitter ──► status/message/artifact_update ──────┼──► browser
│           └ propose_code_change ──► code_expert ──► read_sdk_page ┼──► agent_kb/
└───────┬──────────────────────────┬────────────────────┬───────────┘
        │                          │                    │
        ▼                          ▼                    ▼
┌───────────────┐   ┌──────────────────────┐   ┌──────────────────┐
│ PostgreSQL 17 │   │ AWS S3 (ap-south-1)  │   │ OpenAI API       │
│               │   │                      │   │  gpt-5.3-codex   │
│ users         │   │ {user_id}/projects/  │   │  gpt-5-nano      │
│ projects      │   │   {project_id}/      │   └──────────────────┘
│  ai_state     │   │     head/backtest.py │   ┌──────────────────┐
│  s3_prefix    │   │     pending/         │   │ Market data      │
│ chat_messages │   │     backtests/{id}/  │   │  Zerodha (OAuth) │
│ backtests     │   │       *_trades.csv   │   │  Databento       │
│ token_usage   │   │       *_data.json    │   │  Massive         │
│ data_providers│   │       *_inspector.html│  └──────────────────┘
│ paper_trades  │   │                      │
│ optimizations │   │ presigned URLs, 1h   │   ┌──────────────────┐
└───────────────┘   └──────────────────────┘   │ AWS SES (email)  │
                                                └──────────────────┘
```

### The canonical request flow

```
 1. Browser ──user_message──► WS route
 2. WS route: asyncio.create_task(_process_and_emit)  [own AsyncSession]
 3. AgentLoop pre-flight:  rate limit → token quota → optimistic lock
                              │ duplicate message_id → return cached response
 4. Build session:  ProjectStateSession(ai_state.conversation)
                    wrapped in OpenAIResponsesCompactionSession(gpt-5-nano)
 5. Build ephemeral blocks:  active view · pending changeset · rejection ·
                             is_new_project → session_input_callback
 6. Runner.run_streamed(main_agent)
        │
        ├─► stream event: tool_called ──► emit_status("Editing strategy") ──► browser
        │
        ├─► propose_code_change
        │      └─► code_expert (fresh ephemeral session)
        │             ├ navigator (~350 tok) in the system prompt
        │             ├ read_sdk_page("exit-rules")       ──► emit_status ──► browser
        │             ├ read_sdk_page("template-structure")
        │             └ propose_files ──► SDK validation
        │                    ├ invalid → error returned to Codex → self-heal, retry
        │                    └ valid   → S3 pending/backtest.py
        │                               → emit artifact_update ──► browser diff
        │
        └─► final message text
 7. TokenUsage row written from result.context_wrapper.usage (incl. sub-run)
 8. emit_status(None)  ──► turn complete
 9. asyncio.create_task(_maybe_compact(...))   [after response]
10. Browser: user clicks Accept
        └─► user_event accept_changeset ──► pending/ promoted to head/
11. Later: POST backtest ──► BacktestService downloads head/backtest.py from S3,
           imports the strategy, runs it via quantjo-sdk, writes artifacts to S3,
           stores summary_metrics JSONB in Postgres
12. GET /backtests/{id} ──► metrics from Postgres + freshly minted presigned URLs
```

---

## 4. Deep Technical Explanation

**Why WebSocket for chat, REST for everything else.** The conversation is bidirectional, long-running,
and stateful — progress push, accept/reject, cancellation, message queuing. Project CRUD is none of
those. Using the right protocol per interaction beats forcing one everywhere. See
[07](07_websocket_streaming.md).

**Why Postgres *and* S3.** A backtest emits an interactive HTML inspector that can be megabytes. That
does not belong in a row. But dashboards need to sort by Sharpe ratio without reading object storage,
so summary metrics are **deliberately duplicated** into a JSONB column. That's denormalization for
read performance, with S3 as the source of truth. Being able to name the duplication and justify it is
the point.

**Why presigned URLs, minted per request.** Artifacts are private. Presigned URLs give time-boxed
access without proxying bytes through the API. They expire in an hour — which caused a real bug where
an optimization result tab showed "Forbidden" after an hour. The fix was `GET /optimizations/{id}`
minting fresh URLs on demand rather than storing them. Good concrete example of a lifecycle bug.

**Why the changeset is a path convention, not a table.** `project_artifacts` was dropped. Files live
at `{s3_prefix}/head/backtest.py` and `{s3_prefix}/pending/backtest.py`. One source of truth, no
table/storage divergence, no orphan rows. The cost: no SQL over file metadata — acceptable when each
project has one file.

**Where the transactional boundary isn't.** Postgres and S3 are not in one transaction. An S3 write
followed by a failed DB commit leaves an orphaned pending file. The path convention bounds the damage
— the next proposal overwrites the same key — but it is eventual, not atomic. **Say this before
they ask.**

**Compaction as a context-window strategy.** History would grow unbounded. At 20 non-user items,
`gpt-5-nano` summarizes, in a background task so the user never waits.

**Cost as an architectural concern.** Rate limits, monthly token quotas, model-per-task selection,
prompt caching via a static prefix, the navigator's 96% prompt reduction, compaction, and `max_turns`
are all cost controls. `token_usage` records per-turn input/output/cached tokens for attribution.
**Framing these as one coherent cost architecture is a strong system-design answer.**

---

## 5. My Actual Contribution

**What existed** — the entire system design above.

**What I built** — three components inside it: the knowledge-base compilation + navigator + retrieval
tool on the code-generation path; the streaming progress path from agent events to browser; and the
external regression harness that validates end-to-end agent behaviour over the real WebSocket
protocol.

**What changed** — the most expensive path got ~96% cheaper on always-on tokens, the slowest
interaction became observable, and the least testable component got a test suite.

**Result** — measurable cost reduction, better perceived performance, and a deploy-time quality gate.

---

## 6. Files and Code

See [01](01_company_and_product.md) §6 for the full map. System-design-relevant entry points:

| Concern | Location |
|---|---|
| App composition | `quantjo_api/main.py` |
| Real-time transport | `routes/websocket.py`, `agent/connection_manager.py`, `agent/event_emitter.py` |
| Orchestration | `agent/agent_loop.py` |
| State + idempotency | `agent/services/state_coordinator.py`, `agent/session.py` |
| Cost controls | `agent/services/guardrails.py`, `services/billing.py`, `models/token_usage.py` |
| Storage | `services/s3.py`, `services/head_files.py`, `models/project.py` (`s3_prefix`) |
| Execution | `services/backtest.py` |
| Knowledge base | `agent_kb/`, `agent/tools/read_sdk_page.py` |
| Quality gate | `quantjo-agent-tests/` |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| WS for chat, REST for CRUD | Protocol matched to interaction | Two surfaces to maintain |
| Postgres + S3 split | Queryable metadata, cheap blobs | Not transactional across both |
| Metrics duplicated into JSONB | Dashboards don't touch S3 | Denormalized; can drift |
| Presigned URLs minted per request | Private, time-boxed, no proxying | Expiry bugs if cached |
| Path convention over artifact table | One source of truth | No SQL over file metadata |
| Pending/head changeset | AI edits reviewable | Extra state + UX step |
| Per-user provider credentials | Licensing + blast radius | Onboarding friction |
| Sync backtest execution | Simplicity | Blocks; known debt |
| In-process connection registry | Simple | Blocks horizontal scaling |
| Single AI provider | Focus | Single point of failure |
| ap-south-1 | Latency to Indian markets and users | Single-region DR |

---

## 8. Production Improvements

**Ordered by what I'd actually do first:**

1. **Async backtest execution** — SQS + ECS/Batch workers, notify over the existing WebSocket.
   Removes the worst blocking path.
2. **Redis pub/sub for WebSocket fan-out** — prerequisite for running more than one API instance.
3. **Circuit breaker + retry on OpenAI** — currently a provider blip is a product outage.
4. **Observability** — OpenTelemetry traces spanning HTTP → agent → OpenAI → S3, structured JSON
   logs with request IDs, dashboards for turn latency and per-tool timing.
5. **Automated tests for the REST layer** — the agent has a harness; the API doesn't.
6. **Read replicas** for list endpoints.
7. **CDN for backtest artifacts** instead of presigned URLs per request.
8. **Multi-region DR.**
9. **Per-tenant cost dashboards** from `token_usage`.
10. **Feature flags** on agent prompts for staged rollout.

---

## 9. Common Interview Questions

1. Draw the system.
2. Walk one request end to end.
3. Why WebSocket and REST both?
4. Why two storage systems?
5. How do you keep Postgres and S3 consistent?
6. What's the single point of failure?
7. What breaks first under load?
8. How do you scale to 100x users?
9. How do you handle a 60-second request behind a load balancer?
10. How is a user's data isolated from another's?
11. How do you meter and bill AI usage?
12. What happens if OpenAI goes down?
13. How do you deploy without dropping WebSocket connections?
14. Why ap-south-1?
15. How do you prevent duplicate agent runs?
16. Where does caching happen?
17. How would you add a second AI provider?
18. What's your data retention story?
19. How do you debug a slow turn?
20. What would you redesign?

---

## 10. Strong Answers

**Q5 — Postgres/S3 consistency.**
> They're not consistent in a transactional sense, and I'd rather state that plainly than pretend
> otherwise. A changeset writes the file to S3 and then updates Postgres, and there's no distributed
> transaction. If the S3 write succeeds and the commit fails, you get an orphaned object at the
> pending path. What limits the damage is the path convention: pending files live at a deterministic
> key derived from the project, so the next proposal overwrites the same key rather than accumulating
> garbage. So it's eventually consistent with a bounded failure mode rather than atomic. If I needed
> real guarantees I'd use the outbox pattern — write intent to Postgres in the same transaction as the
> state change, then have a worker perform the S3 operation idempotently and mark it done.

**Q7 — What breaks first?**
> The WebSocket connection registry. `ConnectionManager` holds connections in an in-process
> dictionary, so the moment you run a second API instance, an event originating on instance B can't
> reach a user connected to instance A. For the chat path itself that's survivable because the turn
> runs on the instance holding the socket — but anything server-initiated from elsewhere, like a
> paper-trade signal or a backtest finishing on a worker, silently doesn't arrive. Second is the
> synchronous backtest execution, which pins the process for the duration of a run. Third is the
> database connection pool, since each agent turn holds a session for thirty to sixty seconds — that's
> a very long checkout, so pool exhaustion arrives sooner than the request rate would suggest.

**Q9 — 60-second request behind a load balancer.**
> That's part of why the design uses a WebSocket rather than a long HTTP request. Load balancers and
> proxies have idle timeouts — often 60 seconds — and a long POST that sends nothing until it finishes
> is exactly the shape that gets killed. The WebSocket sends status frames every few seconds as tools
> fire, so the connection is never idle, which keeps intermediaries happy and gives the user progress
> at the same time. The same property is why the backtest path is the weaker one: it *is* a long
> synchronous HTTP request, and it's the piece I'd move to a queue first.

**Q20 — What would you redesign?**
> Two things. Backtests would be asynchronous from the start — a job row, a queue, workers, and
> completion pushed over the WebSocket that's already open. That single change removes the blocking
> path, gives you retries, and makes the system horizontally scalable for the heaviest workload.
> Second, I'd externalize the WebSocket connection registry to Redis from day one. It's a small amount
> of work early and a painful migration later, and it's the thing standing between this and running
> more than one instance. I would not change the Postgres/S3 split or the changeset model — those have
> held up well.

---

## 11. Follow-up Questions

- "Each agent turn holds a DB session for 60 seconds. What's your pool sizing math?"
- "How do you deploy without killing in-flight agent turns?"
- "Two tabs open on one project — what does the second one see?"
- "`ai_state` is JSONB holding the whole conversation. What's the row size ceiling?"
- "How would you shard if one user had 100,000 projects?"
- "What's your RPO and RTO?"
- "Where would you put a cache, and what would you cache?"

---

## 12. Senior Engineer Perspective

Senior engineers evaluate a system design on **failure modes, boundaries, and cost** — not features.

- **Failure modes:** single AI provider, no circuit breaker, in-process connection registry,
  synchronous backtests, non-transactional Postgres/S3 boundary. Name all five unprompted.
- **Boundaries:** the SDK-only dependency rule and the pending/head changeset are both clean
  boundaries that make the system easier to reason about. Highlight them.
- **Cost:** this is an unusual system in that *inference cost per interaction* is a first-order design
  constraint. Presenting rate limits, quotas, model selection, prompt caching, the navigator, and
  compaction as one coherent cost architecture is exactly the framing a staff engineer would use.
- **Evolution:** they'll want to know what you'd do at 10x and 100x, and that the answers differ. See
  [11](11_scalability.md).

---

## 13. Common Mistakes

- **Listing components instead of tracing a request.** Always trace.
- **Claiming you designed it.** You built three components inside it.
- **Ignoring cost.** For an AI product this is a first-class design axis, and forgetting it marks you
  as inexperienced with production LLM systems.
- **Not knowing what breaks first.** Every system-design interview asks. Have the ordered list.
- **Pretending the storage boundary is transactional.** It isn't. Own it.

---

## 14. Cheat Sheet

**Flow:** Browser → WS → FastAPI → pre-flight (rate limit · quota · idempotency lock) → session +
ephemeral context → `Runner.run_streamed` → tools → `propose_code_change` → Codex → navigator →
`read_sdk_page` → `propose_files` → SDK validation → S3 `pending/` → `artifact_update` → accept →
`head/` → backtest → S3 artifacts + Postgres metrics → presigned URLs.

**Storage split:** Postgres = queryable metadata (`summary_metrics` JSONB duplicated for dashboards).
S3 = strategy files + trade CSV + JSON + multi-MB inspector HTML. Presigned, 1h, minted per request.

**Cost architecture:** rate limit · monthly token quota · model per task · static cacheable prompt ·
navigator (−96%) · compaction at 20 non-user items · `max_turns=10` · `token_usage` per turn.

**Breaks first (ordered):** in-process `ConnectionManager` → sync backtests → DB pool (60s checkouts)
→ single-region → single AI provider.

**Non-transactional boundary:** Postgres ↔ S3. Bounded by path convention; fix = outbox pattern.

**Region:** ap-south-1 (NSE/NFO latency). Single-region DR today.
