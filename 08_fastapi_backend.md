# FastAPI Backend

---

## 1. Interview Answer (2–3 minutes)

> The backend is FastAPI on Python 3.12, fully async — SQLAlchemy 2.0 with asyncpg against PostgreSQL
> 17, Pydantic v2 for validation, about 54 REST endpoints plus one WebSocket endpoint. The layering is
> conventional: `routes/` holds thin HTTP handlers, `services/` holds business logic, `models/` is the
> SQLAlchemy ORM, `schemas/` is the Pydantic request/response contract, and `core/` has the database
> engine and security primitives.
>
> Async matters here more than in a typical CRUD app because of the workload shape. A single agent
> turn spends thirty to sixty seconds waiting on OpenAI, plus S3 round trips and database queries.
> That's almost all I/O wait. With a synchronous framework each of those turns would occupy a worker
> thread doing nothing; with asyncio the event loop handles hundreds of concurrent turns on one
> process because they're all awaiting.
>
> Dependency injection is where FastAPI is genuinely good. `get_db` yields a session per request and
> closes it in a finally block, `get_current_user` decodes the JWT and loads the user, and route
> handlers just declare what they need as parameters. It's constructor injection driven by type hints,
> and it makes handlers testable because you can override a dependency in tests.
>
> The one place I had to deliberately break the request-scoped session pattern was the WebSocket path.
> The agent turn runs as a background asyncio task while the receive loop keeps reading frames, and
> `AsyncSession` isn't safe for concurrent coroutine use — sharing one corrupts SQLAlchemy's identity
> map. So each turn opens its own session. That's the most interesting backend detail I worked on.

---

## 2. What Problem Were We Solving?

**Business.** A multi-tenant SaaS needs auth, ownership isolation, quotas, and billing enforcement on
every path — and it needs to hold long-running AI conversations without falling over.

**Technical.**
- **I/O-bound, long-lived requests.** Agent turns are 30–60s of mostly waiting.
- **Mixed protocols.** REST for CRUD, WebSocket for the conversation.
- **Strict contracts.** The frontend is a separate TypeScript codebase; response shapes must be
  precise and documented.
- **Multi-tenancy.** Every query must be scoped by `user_id`. A missing filter is a data breach.

**Why not Django or Flask?** Django brings an ORM, admin, and sync-first request model we'd fight —
we need async SQLAlchemy and there's no admin use case. Flask needs extensions for validation, async,
and docs, all assembled by hand. FastAPI gives native async, Pydantic validation, DI, and OpenAPI
generation out of the box — and OpenAPI matters because the frontend consumes the contract directly.

---

## 3. Architecture

```
                       HTTP / WS
                            │
              ┌─────────────▼──────────────┐
              │  main.py — FastAPI app     │
              │  CORS, routers, /health    │
              └─────────────┬──────────────┘
                            ▼
   ┌────────────────────────────────────────────────────┐
   │ routes/                                            │
   │  auth · users · projects · backtests · providers   │
   │  assets · templates · optimizations · paper_trades │
   │  websocket                                         │
   │                                                    │
   │  Depends(get_db) ──────────► AsyncSession          │
   │  Depends(get_current_user) ─► User                 │
   │  Pydantic schema in/out ────► validation + OpenAPI │
   └───────────────────────┬────────────────────────────┘
                           ▼
   ┌────────────────────────────────────────────────────┐
   │ services/    business logic, no HTTP knowledge     │
   │  ProjectService · BacktestService · AuthService    │
   │  ProviderService · PaperTradingService             │
   │  billing · head_files · s3                         │
   └───────────────────────┬────────────────────────────┘
                           ▼
   ┌──────────────────────────────┐   ┌─────────────────┐
   │ models/  SQLAlchemy ORM      │   │ agent/          │
   │ UUID PKs, JSONB, soft delete │   │ the AI layer    │
   └──────────────┬───────────────┘   └─────────────────┘
                  ▼
   PostgreSQL 17 (asyncpg)      +      S3 (boto3)
```

**REST request flow:** request → CORS → router match → Pydantic validates body → `Depends` resolves
`get_db` and `get_current_user` → handler → service → ORM → DB → response model serializes → JSON.

**WebSocket flow:** see [07](07_websocket_streaming.md).

---

## 4. Deep Technical Explanation

### Dependency injection

FastAPI's DI is driven by type hints and `Depends`:

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

The generator form matters: code before `yield` is setup, code after (or the `async with` exit) is
teardown that runs even if the handler raises. That's how session cleanup is guaranteed.

Dependencies compose — `get_current_user` itself depends on `get_db` — and FastAPI caches a
dependency's result within a single request, so `get_db` resolving twice in one request yields the
same session. That's what makes a request a single transactional unit.

**Why it's better than a module-level singleton:** it's overridable. `app.dependency_overrides[get_db]
= lambda: test_session` swaps the database in tests with no monkeypatching. Explicit dependencies also
make a handler's requirements readable from its signature.

### Async, and where it actually pays

The workload is ~95% I/O wait: OpenAI (tens of seconds), S3, Postgres. In a sync framework each
request pins a worker; concurrency is bounded by thread-pool size. With asyncio, an awaiting
coroutine costs a small amount of memory and no thread, so one process handles far more concurrent
agent turns.

**The classic failure mode to name:** a single blocking call in an async handler blocks the *entire*
event loop, not just that request. Every I/O path must be async — `asyncpg` not `psycopg2`, `httpx`
not `requests`. `psycopg2-binary` is in the dependency list but only for Alembic migrations, which
are synchronous by design. Boto3 is sync and is a real wart on this path; production would use
`aioboto3` or push it to a thread.

### Session lifecycle — and the WebSocket exception

REST: one session per request, via `Depends(get_db)`.

WebSocket: **one session per turn**, deliberately. `_process_and_emit` opens its own
`AsyncSessionLocal()` because the agent turn is a background task running concurrently with the
receive loop, and `AsyncSession` is not coroutine-safe. Sharing one produced sporadic "Instance is not
persistent" errors from identity-map corruption. See [07](07_websocket_streaming.md) §4 and
[14](14_my_contributions.md) — this is your best debugging story.

Related: `agent_loop.py` uses `await db.get(Project, project.id)` rather than `db.refresh(project)`,
because `refresh` raises on a detached instance (which happens if a tool rolled back), while `get` is
identity-map-aware.

### Background tasks

Two mechanisms in play, and knowing the difference is a good signal:

- **`asyncio.create_task`** for the agent turn — needs to run concurrently with the receive loop so
  the connection stays responsive to cancellation and queued messages. Also used for compaction:
  `asyncio.create_task(_maybe_compact(session, conversation))` runs summarization *after* the response
  is returned, so the user isn't waiting on it.
- **FastAPI `BackgroundTasks`** would run after the response is sent on a REST path — the right tool
  for fire-and-forget work like sending an email.

**The honest limitation:** both are in-process. If the process dies mid-task, the work is lost — no
retry, no durability, no visibility. Backtests currently run synchronously and block the request,
which is documented as a known limitation. Production needs a real queue (SQS/Celery) with workers,
retries, and a dead-letter queue.

### Error handling

Three layers:

1. **Pydantic validation** → automatic 422 with field-level detail, before your code runs.
2. **`HTTPException`** with structured bodies. The newer auth surface uses
   `detail = {code, message}` with machine-readable codes (`email_already_registered`,
   `email_not_verified`, `use_google_signin`, `invalid_credentials`) so the frontend can branch UX
   rather than string-matching English. Same pattern on paper-trade gating
   (`no_strategy | no_head_backtest | already_active`).
3. **Domain exceptions** — `GuardrailError` carries `error_type` and `code`; `billing.QuotaExceeded`
   carries `meter`, `used`, `limit`, `resets_at`, `plan` so the client can render a real upgrade
   prompt.

On the WebSocket there are no status codes, so errors are `{"type":"error","code":...,"message":...}`
frames via `emit_error`, and the agent loop catches broadly and returns a user-facing message rather
than dropping the connection.

### Multi-tenancy

Every project-scoped query filters on `user_id` — `ProjectService.get_project_by_id(db, project_id,
user_id)` takes the user as a required argument rather than fetching by ID and checking afterward.
UUID PKs mean IDs aren't guessable, but that's defense in depth, not the control. S3 keys are prefixed
by `{user_id}/` so the storage layout mirrors the isolation boundary.

---

## 5. My Actual Contribution

**What existed** — the entire FastAPI application: app setup, routers, DI, models, schemas, services,
auth, S3 integration, migrations.

**What I built / changed** — work inside `agent/`: the streaming event handler, the navigator and
`read_sdk_page` tool, the `TOOL_LABELS` map, and the ephemeral-vs-persisted status distinction. On
the infrastructure side I was involved in the per-turn session fix on the WebSocket path.

**What changed** — the agent path became observable and cheaper; a class of intermittent
identity-map errors was eliminated.

**Result** — no more sporadic "Instance is not persistent" failures under concurrent turns.

> **Be careful here.** This is the file where over-claiming is easiest and most damaging. You worked
> *within* a FastAPI codebase; you didn't architect it. Know it well, claim precisely.

---

## 6. Files and Code

| Path | Contents |
|---|---|
| `quantjo_api/main.py` | App instance, CORS, router registration, `/health` |
| `quantjo_api/config.py` | Pydantic `Settings`, `.env` loading |
| `quantjo_api/core/database.py` | Async engine, `AsyncSessionLocal`, `get_db` |
| `quantjo_api/core/security.py` | JWT create/decode, bcrypt via passlib |
| `quantjo_api/routes/` | `auth.py`, `users.py`, `projects.py`, `backtests.py`, `providers.py`, `assets.py`, `templates.py`, `optimizations.py`, `paper_trades.py`, `websocket.py` |
| `quantjo_api/services/` | `project.py`, `backtest.py`, `auth.py`, `provider.py`, `paper_trading.py`, `billing.py`, `head_files.py`, `s3.py` |
| `quantjo_api/models/` | `user.py`, `project.py`, `chat_message.py`, `backtest.py`, `token_usage.py`, `data_provider.py`, `refresh_token.py`, `password_reset_token.py`, `email_verification_token.py` |
| `quantjo_api/schemas/` | Pydantic request/response models |
| `quantjo_api/agent/services/` | `guardrails.py`, `state_coordinator.py`, `changeset_handler.py`, `debug_logger.py` |
| `alembic/versions/` | Migrations |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| FastAPI | Native async, Pydantic, DI, OpenAPI | Smaller ecosystem than Django |
| Fully async SQLAlchemy 2.0 + asyncpg | Workload is I/O-bound | One blocking call stalls the loop |
| Session per request (REST) | Clear transactional unit | — |
| Session per turn (WS) | `AsyncSession` isn't coroutine-safe | Extra connection per turn |
| Services separate from routes | Testable, protocol-agnostic | More files |
| Pydantic schemas ≠ ORM models | API contract decoupled from storage | Duplication between layers |
| Structured error bodies | Frontend branches on codes, not English | More verbose than a bare string |
| Sync backtest execution | Simplicity first | Blocks a worker; documented limitation |
| `asyncio.create_task` for background | Concurrency with the receive loop | No durability or retry |
| UUID PKs + explicit `user_id` filters | Multi-tenant isolation | Larger indexes |

---

## 8. Production Improvements

1. **Move backtests to a queue** — SQS + Batch/ECS workers, notify over the existing WebSocket. The
   single biggest architectural debt.
2. **Replace boto3 with `aioboto3`** or offload to a thread — sync S3 calls in async handlers stall
   the loop.
3. **Durable background jobs** — Celery/Arq with retries and a DLQ, replacing `create_task`.
4. **Connection-pool tuning and observability** — pool size, checkout time, saturation alerts.
5. **Structured JSON logging + request IDs** propagated through the agent turn; there are still
   `print()` calls in `connection_manager.py`.
6. **OpenTelemetry tracing** — a trace spanning HTTP → agent → OpenAI → S3 would make the 30–60s turn
   attributable.
7. **Automated tests.** `CLAUDE.md` says "Manual Testing Only — No automated tests" for the API. The
   agent harness exists; the API layer has none.
8. **Read replicas** for list endpoints.
9. **Rate limiting at the edge** (API Gateway/nginx) in addition to in-process guardrails.
10. **Idempotency keys on mutating REST endpoints**, matching the `message_id` pattern already used
    on the agent path.

---

## 9. Common Interview Questions

1. Why FastAPI over Django or Flask?
2. How does dependency injection work in FastAPI?
3. Why is `get_db` a generator?
4. What does `async` buy you here?
5. What happens if you make a blocking call in an async handler?
6. How is a database session scoped?
7. Why does the WebSocket path use a different session strategy?
8. How do you handle background work?
9. Difference between `asyncio.create_task` and `BackgroundTasks`?
10. How do you handle errors and what does the client see?
11. Why structured error codes instead of messages?
12. How is multi-tenancy enforced?
13. Why separate Pydantic schemas from ORM models?
14. How do migrations work, and why is `psycopg2` present in an async app?
15. What's in `core/` and why?
16. How would you test a route handler?
17. What's your connection pool sizing strategy?
18. How does OpenAPI documentation get generated?
19. What's the biggest architectural debt in this backend?
20. How would you scale this to 100x traffic?

---

## 10. Strong Answers

**Q5 — Blocking call in an async handler.**
> It blocks the entire event loop, not just that request — every other in-flight request on that
> process stalls until it returns. It's the classic async footgun and it's insidious because it looks
> fine under light load and collapses under concurrency. That's why the stack is asyncpg rather than
> psycopg2 and httpx rather than requests. The honest gap in our codebase is boto3, which is
> synchronous — every S3 call in an async path is a small stall. It's tolerable because the calls are
> short and the dominant latency is OpenAI, but the correct fix is `aioboto3` or pushing those calls
> to a thread pool with `run_in_executor`. psycopg2 is in our dependencies, but only for Alembic,
> which runs migrations synchronously outside the request path.

**Q7 — Different session strategy on WebSocket.**
> Because concurrency is different there. On REST, one request equals one task equals one session —
> no sharing. On the WebSocket, the agent turn runs as a background asyncio task so the receive loop
> stays responsive to cancellation and queued messages. If both share the connection-level
> `AsyncSession`, two coroutines interleave on an object that isn't safe for that, and SQLAlchemy's
> identity map gets corrupted. The symptom was sporadic "Instance is not persistent" errors that
> wouldn't reproduce reliably because they were timing-dependent. The fix is one `AsyncSessionLocal()`
> per turn, with a fresh `AgentContext` bound to it so all tools in a turn share one transaction. The
> underlying principle is that a session is a unit of work, and concurrent units of work need separate
> sessions.

**Q11 — Structured error codes.**
> Because the frontend needs to branch on the error, and branching on English is fragile. If login
> fails, the UI does something different for "email not verified" — show a resend link — versus
> "you signed up with Google" — show the Google button — versus "wrong password". With a plain string
> the frontend either string-matches, which breaks the moment someone rewords the copy, or shows a
> generic message. So the auth surface returns `detail: {code, message}` with codes like
> `email_not_verified` and `use_google_signin`. Same idea for quota errors, which carry `used`,
> `limit`, `resets_at`, and `plan` so the client can render a real upgrade prompt instead of "quota
> exceeded".

**Q19 — Biggest architectural debt.**
> Synchronous backtest execution. A backtest runs inside the request, so it blocks, there's no queue,
> no retry, and no way to run more than a handful concurrently. It's documented in the repo as a known
> limitation with AWS Batch as the intended fix. The right shape is: the endpoint creates a row with
> status `pending`, enqueues to SQS, returns immediately, a worker picks it up, writes results to S3
> and updates the row, and the user gets notified over the WebSocket they already have open. That also
> gives you retries and a dead-letter queue for free. Second place would be the lack of automated
> tests on the API layer — the agent harness I built covers agent behaviour, but the REST surface is
> tested by hand through Postman collections.

---

## 11. Follow-up Questions

- "Dependencies are cached per request. What breaks if they weren't?"
- "You use `db.get` instead of `db.refresh` in the agent loop. Why specifically?"
- "What's your pool size, and what happens when it's exhausted mid-agent-turn?"
- "JSONB for `ai_state` and `summary_metrics` — when does that become a problem?"
- "Soft delete via `deleted_at` — how do you stop a stale query from leaking deleted rows?"
- "How do you handle a migration that needs to run against a live table?"
- "What's the blast radius of a missing `user_id` filter?"

---

## 12. Senior Engineer Perspective

They'll assess whether you understand the framework or just used it. The distinguishing knowledge:
**the generator form of `get_db` guarantees teardown**, **dependency caching makes a request one
transactional unit**, **a single blocking call stalls everything**, and **session scope must follow
concurrency, not convention** — which is exactly the WebSocket insight.

They'll also probe consistency boundaries: Postgres and S3 aren't in one transaction, so a
partially-failed changeset write is possible. The path convention limits the damage (the next
proposal overwrites) but it's not transactional and you should say so.

And they'll want to know what you'd fix. "Sync backtests, sync boto3, no API tests, in-process
background work" — naming your own architecture's debt is a senior habit.

---

## 13. Common Mistakes

- **Claiming you built the backend.** You worked in it. Say so.
- **"Async makes it faster."** Async improves *concurrency* for I/O-bound work. It does nothing for
  CPU-bound work and can make it worse.
- **Not knowing why `get_db` yields.** It's the most-asked FastAPI DI question.
- **Confusing `BackgroundTasks` with `create_task`.** Different lifetimes, different use cases.
- **Ignoring the sync/async boundary.** boto3 is sync in an async app. Know it; it's the kind of
  detail that shows you read the code.

---

## 14. Cheat Sheet

- **Stack:** FastAPI · Python 3.12 · SQLAlchemy 2.0 async · asyncpg · Pydantic v2 · PostgreSQL 17 ·
  boto3/S3 · Alembic. ~54 REST + 1 WS.
- **Layers:** `routes/` (HTTP) → `services/` (logic) → `models/` (ORM) · `schemas/` (contracts) ·
  `core/` (db, security) · `agent/` (AI).
- **DI:** `Depends(get_db)` — generator ⇒ guaranteed teardown; cached per request ⇒ one session per
  request. `Depends(get_current_user)` — JWT → User. Overridable in tests.
- **Async pays** because ~95% of an agent turn is I/O wait. One blocking call stalls the whole loop.
- **Sessions:** per-request on REST; **per-turn** on WS (`AsyncSession` is not coroutine-safe →
  identity-map corruption → "Instance is not persistent").
- **`db.get` over `db.refresh`** — identity-map-aware, survives a detached instance after rollback.
- **Background:** `asyncio.create_task` (agent turn, compaction) vs `BackgroundTasks` (post-response).
  Neither is durable.
- **Errors:** Pydantic 422 · `HTTPException` with `detail={code,message}` · `GuardrailError` ·
  `QuotaExceeded(meter, used, limit, resets_at, plan)` · WS `error` frames.
- **Multi-tenancy:** `user_id` on every query · UUID PKs · S3 prefixed `{user_id}/`.
- **Debt:** sync backtests · sync boto3 · no API tests · in-process background work.
