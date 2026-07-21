# Scalability

---

## 1. Interview Answer (2–3 minutes)

> The honest starting point is that this was a public beta running on a single instance, so most of
> what I'd say about scaling is analysis rather than something I load-tested. But I can tell you
> precisely what breaks first and in what order, because the bottlenecks are structural and visible in
> the code.
>
> The first thing that breaks is the WebSocket connection registry. It's an in-process dictionary
> mapping project ID to a set of connections. The moment you run a second API instance, an event
> originating on instance B can't reach a user connected to instance A. Chat still works, because the
> agent turn runs on the same instance holding the socket — but anything server-initiated from
> elsewhere silently doesn't arrive. The fix is standard: Redis pub/sub with a channel per project,
> each instance subscribing for its local connections.
>
> Second is synchronous backtest execution. A backtest runs inside the request and pins the process
> for its duration. That needs to be a queue and workers.
>
> Third is the database connection pool, and this one is subtle. Each agent turn holds a session for
> thirty to sixty seconds because the turn is mostly waiting on OpenAI. That's an extraordinarily long
> checkout. A pool of twenty connections supports roughly twenty concurrent turns regardless of how
> fast your queries are — you hit pool exhaustion at a request rate far below what the database could
> actually handle.
>
> The unusual thing about scaling this system is that the binding constraint isn't CPU or database
> throughput — it's OpenAI rate limits and cost. Ten thousand concurrent users means ten thousand
> concurrent inference requests, and no amount of horizontal scaling on my side changes that. Which is
> why the token optimization work I did is a scalability contribution, not just a cost one.

---

## 2. What Problem Were We Solving?

**Business.** Beta growth would mean more concurrent conversations, more backtests, and linearly
growing inference spend. Unit economics have to hold as volume grows, or growth is a liability.

**Technical.** The workload is unusual:
- Requests are **long** (30–60s) and **I/O-dominated**
- Connections are **stateful and persistent**
- The expensive resource is a **third-party API with its own rate limits**
- Artifacts are **large** (multi-MB HTML)
- Cost per request is **high and variable**

**Why standard scaling advice doesn't fully apply.** "Add more instances behind a load balancer" fails
for stateful WebSocket connections without a shared registry, and it doesn't help at all when the
bottleneck is a vendor's rate limit.

---

## 3. Architecture — where the limits are

```
        ┌────────────────────────────────────────────┐
        │  BROWSER × N                               │
        └──────────────────┬─────────────────────────┘
                           │ persistent WS connections
                  ┌────────▼─────────┐
                  │  Load Balancer   │  ◄── needs sticky sessions
                  └────────┬─────────┘      or connection routing
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
     ┌───────────┐  ┌───────────┐  ┌───────────┐
     │ FastAPI 1 │  │ FastAPI 2 │  │ FastAPI 3 │
     │           │  │           │  │           │
     │ ⛔ Connection Manager is IN-PROCESS      │  ◄── BOTTLENECK #1
     │    instance 2 cannot notify a user       │
     │    connected to instance 1               │
     │                                          │
     │ ⛔ backtests run SYNCHRONOUSLY here       │  ◄── BOTTLENECK #2
     └─────┬─────────────┬──────────────┬───────┘
           │             │              │
           ▼             ▼              ▼
   ┌──────────────┐  ┌────────┐  ┌──────────────────┐
   │ PostgreSQL   │  │  S3    │  │  OpenAI API      │
   │              │  │        │  │                  │
   │ ⛔ pool: each │  │ scales │  │ ⛔ RATE LIMITS +  │
   │ agent turn   │  │ fine   │  │    COST are the  │
   │ holds a conn │  │        │  │    real ceiling  │
   │ 30–60s       │  │        │  │                  │
   │ BOTTLENECK #3│  │        │  │  BOTTLENECK #4   │
   └──────────────┘  └────────┘  └──────────────────┘
```

---

## 4. Deep Technical Explanation

### Bottleneck 1 — in-process connection registry

```python
class ConnectionManager:
    def __init__(self):
        self._connections: Dict[UUID, Set[ConnectionState]] = {}
        self._lock = asyncio.Lock()
```

Process-local. Consequences at N > 1 instances:
- `broadcast_to_project` only reaches connections on *this* instance.
- A worker finishing a backtest on another machine can't notify the user.
- `get_project_connection_count` returns a local count, not a global one.

**Fix:** Redis pub/sub, channel per project. Each instance subscribes for its local connections and
publishes on emit. Add sticky sessions at the LB so a reconnect lands on an instance that can serve
it, and externalize presence (`SETEX` per connection) so "is this user online" is answerable globally.

**Cost:** an extra network hop per emit and a Redis dependency in the critical path. Acceptable — but
note it introduces at-most-once delivery semantics unless you add acknowledgment.

### Bottleneck 2 — synchronous backtests

Documented in the repo as a known limitation. A backtest downloads code from S3, dynamically imports
it, runs it through the SDK over historical market data, and writes artifacts — all inside the
request. This pins a worker, makes the request vulnerable to LB idle timeouts, and offers no retry.

**Fix:** the standard job pipeline. `POST` creates a row with status `pending` and enqueues to SQS,
returning immediately. ECS/Batch workers consume, execute, write to S3, update the row, and publish
completion — which the user receives over the WebSocket they already have open. You gain retries, a
dead-letter queue, independent worker scaling, and the ability to size compute to the job rather than
to the web tier.

### Bottleneck 3 — connection pool under long checkouts

This is the most interesting one because it's counterintuitive.

```
Normal API:   query takes 5ms   → 20 connections ≈ 4,000 req/s
This system:  turn holds 45s    → 20 connections ≈ 20 concurrent turns
```

The pool is sized for *concurrency of held sessions*, not query throughput. Twenty concurrent traders
mid-conversation exhausts a 20-connection pool while the database sits nearly idle.

**Two fixes, and the second is better:**

1. **Enlarge the pool** — treats the symptom, and Postgres has its own connection ceiling (each
   backend is a process). PgBouncer in transaction mode helps but interacts badly with session-level
   state.
2. **Don't hold the session across the LLM call.** The turn only needs the DB at the start (load
   project, check quota, acquire lock) and at the end (persist response, write token usage). The
   30–60 seconds in between is pure OpenAI wait, and holding a connection through it is waste.
   Restructuring to acquire → release → await → re-acquire would cut pool pressure by an order of
   magnitude.

**That second answer is the one that distinguishes a strong candidate.** It shows you looked at *why*
the resource is held rather than reaching for a bigger number.

### Bottleneck 4 — the vendor ceiling

This is the real constraint and the one your own architecture can't scale away.

- **Rate limits** — requests and tokens per minute per organization. At high concurrency you queue or
  get 429s regardless of how many instances you run.
- **Cost** — every turn costs real money. 10,000 daily active users × a few turns each × tokens per
  turn is a number that dominates infrastructure spend entirely.
- **Latency** — 30–60s is mostly the model. You cannot optimize that from your side.

**Mitigations, in order of leverage:**

| Mitigation | Effect | Status |
|---|---|---|
| Reduce tokens per request | Directly raises TPM headroom and cuts cost | **Done** — the navigator, ~96% off the code path |
| Prompt caching | Cheaper cached input tokens | **Done** — static prompt prefix |
| Conversation compaction | Bounds context growth | **Done** — 20 non-user items |
| Model per task | nano for compaction, mini for judging | **Done** |
| Per-user quotas + rate limits | Bounds worst-case spend per tenant | **Done** |
| Semantic caching of identical requests | Skips inference entirely | Not built |
| Request queuing with fair scheduling | Graceful degradation instead of 429s | Not built |
| Multiple provider accounts / providers | Raises the ceiling | Not built |

**This is why the token optimization is a scalability story, not just a cost story.** Cutting prompt
size ~96% on the code path directly increases how many concurrent code generations fit under a
tokens-per-minute limit. Frame it that way in a system-design round — it lands much harder.

### What scales fine

- **S3** — effectively unlimited; the access pattern is small and key-distributed.
- **Stateless REST endpoints** — trivially horizontal once the pool issue is addressed.
- **Postgres reads** — list endpoints are ideal for read replicas.
- **The agent's CPU cost** — negligible; it's all I/O wait.

---

## 5. My Actual Contribution

**What existed** — the single-instance architecture and its limits.

**What I built that affects scalability** — the token optimization, which is the direct lever on the
binding constraint. Cutting always-on prompt tokens ~96% on the code path raises effective throughput
under a TPM limit and reduces per-turn cost, which is what makes growth affordable.

**What I did not do** — load testing, capacity planning, Redis fan-out, or the job queue. **Say this.**

**Result** — meaningful headroom on the one resource that couldn't be scaled horizontally.

---

## 6. Files and Code

| Bottleneck | Location |
|---|---|
| Connection registry | `agent/connection_manager.py` — `_connections: Dict[UUID, Set[ConnectionState]]` |
| Sync backtests | `services/backtest.py`, `routes/backtests.py` |
| Session lifetime | `routes/websocket.py::_process_and_emit`, `core/database.py` |
| Rate limiting | `agent/services/guardrails.py` |
| Quotas | `services/billing.py`, `models/token_usage.py` |
| Compaction | `agent/agent_loop.py` — `COMPACTION_THRESHOLD`, `_maybe_compact` |
| Turn bounding | `agent/agent_loop.py` — `max_turns=10` |
| Token reduction | `agent/system_prompt.py`, `agent/tools/read_sdk_page.py`, `agent_kb/` |

---

## 7. Design Decisions

| Decision | Why then | Cost at scale |
|---|---|---|
| In-process connection registry | Simple, no dependency, correct for one instance | Blocks horizontal scaling |
| Sync backtests | Ship the feature | Pins workers; LB timeout risk |
| Session held across the whole turn | Simple transactional story | Pool exhaustion at low concurrency |
| Single AI provider | Focus | Vendor rate limit is the ceiling |
| Single region | Latency to Indian markets | No DR |
| JSONB conversation in a row | No join, simple | Row growth; rewrite on every turn |
| Presigned URLs per request | Private + no proxying | Repeated signing cost |

**The general point worth making:** none of these were wrong decisions. They were correct for a beta
with one instance, and each has a known migration path. Premature distribution would have cost more
than it saved. What matters is knowing *when* each becomes wrong.

---

## 8. Production Improvements

**At 10x (~hundreds of concurrent users):**
1. Redis pub/sub for WebSocket fan-out + sticky sessions.
2. Move backtests to SQS + worker fleet.
3. Restructure the agent turn to release the DB session across the LLM call.
4. Read replicas for list endpoints.
5. Real observability — OTel traces, turn-latency histograms, pool saturation alerts.

**At 100x (~thousands concurrent):**
6. Request queue with fair scheduling per tenant; degrade gracefully instead of 429.
7. Multiple OpenAI accounts / a second provider; route by availability.
8. Semantic caching on near-identical generations.
9. CDN for backtest artifacts.
10. Partition `token_usage` and `chat_messages` by time; archive cold data.
11. Move conversation out of the `projects` row — every turn rewrites a growing JSONB blob.

**At 1000x:**
12. Multi-region with data residency awareness.
13. Dedicated inference capacity or a self-hosted model for the cheapest tasks.
14. Separate the agent tier from the API tier so they scale independently.

---

## 9. Common Interview Questions

1. What breaks first?
2. How do you scale WebSockets horizontally?
3. Why does the connection pool exhaust so quickly?
4. How would you make backtests scale?
5. What's the ceiling you can't engineer around?
6. How does token optimization relate to scalability?
7. Where would you add caching?
8. How do you handle OpenAI rate limits?
9. What's your capacity per instance?
10. How would you load test this?
11. What's the cost curve as users grow?
12. How do you deploy without dropping connections?
13. Read replicas — which queries?
14. When would you shard the database?
15. How do you scale the vector database? *(Trap — there isn't one.)*
16. What's the memory cost per WebSocket connection?
17. How do you prevent one tenant from starving others?
18. What would you monitor first?
19. What's the difference between scaling to 10x and 100x here?
20. What did you actually measure versus infer?

---

## 10. Strong Answers

**Q3 — Pool exhaustion.**
> Because the session is held for the whole agent turn, and the turn is thirty to sixty seconds of
> mostly waiting on OpenAI. In a normal API a connection is checked out for a few milliseconds, so
> twenty connections serve thousands of requests per second. Here twenty connections serve about
> twenty concurrent conversations, full stop — the database is nearly idle while the pool is saturated.
> The naive fix is a bigger pool, but Postgres backends are processes and that has its own ceiling.
> The better fix is structural: the turn only touches the database at the start, to load the project
> and check quota and take the lock, and at the end, to persist the response and write token usage.
> Everything in between is pure network wait. Releasing the session across the LLM call and
> re-acquiring afterward would cut pool pressure by an order of magnitude. It costs you a single
> transaction spanning the turn, which we don't actually need.

**Q5 — The ceiling you can't engineer around.**
> OpenAI. Rate limits and cost, both. At high concurrency you're bounded by tokens per minute at the
> organization level, and no amount of horizontal scaling on my side changes that — I'd just be
> queueing faster. And cost scales linearly with usage in a way that dominates infrastructure spend
> entirely; the compute to run FastAPI is rounding error next to the inference bill. The only real
> levers are using fewer tokens per request, caching, choosing cheaper models where quality allows,
> and eventually multiple accounts or providers. That's exactly why I frame the prompt optimization
> work as a scalability contribution — cutting the code path's always-on prompt by about 96% directly
> increases how many concurrent generations fit under a TPM limit.

**Q15 — Scaling the vector database.**
> There isn't one — we don't use a vector database. Retrieval is exact-ID lookup against fifteen
> markdown files on disk, which is an O(1) filesystem read and scales trivially; if it ever mattered
> I'd load all fifteen into memory at import, since it's about ten thousand tokens total. If you're
> asking what I'd do had we built one: the interesting scaling questions there are index size versus
> memory, replication for read throughput, and rebuild strategy when embeddings change — a full
> re-embed of a large corpus is expensive and you want it to be a background migration rather than
> downtime.

**Q20 — Measured vs inferred.**
> Almost all inferred, and I want to be straight about that. I measured prompt token counts, though
> with a `chars/4` heuristic rather than real tokenization. I did not load test, I did not measure
> connections per instance, and I did not profile pool saturation. What I have is structural analysis
> — I can point at the in-process dictionary in `connection_manager.py` and tell you exactly why a
> second instance breaks, and I can do the arithmetic on session hold time versus pool size. But
> that's reasoning from the code, not from a benchmark, and those are different levels of confidence.
> If I were taking this to production the first thing I'd do is load test to find where the analysis
> is wrong, because it usually is somewhere.

---

## 11. Follow-up Questions

- "You'd add Redis pub/sub. What are the delivery guarantees, and does that matter here?"
- "Sticky sessions plus autoscaling — what happens when an instance is drained?"
- "Releasing the DB session mid-turn loses transactional atomicity. What breaks?"
- "`ai_state` JSONB is rewritten every turn. What's the write amplification?"
- "How do you fairly schedule when three tenants are saturating your TPM limit?"
- "What's your per-connection memory, and how did you arrive at it?"
- "Would you scale the agent tier separately from the API tier? Why?"

---

## 12. Senior Engineer Perspective

**The key signal is ordering.** Anyone can list bottlenecks. A senior engineer knows which binds
first, why, and what the *right* fix is versus the expedient one — the pool answer is the clearest
example: bigger pool versus don't hold the connection.

**They'll want to hear that the constraint is external.** Recognizing that OpenAI's rate limits and
cost dominate, and that this reframes prompt optimization as a throughput lever, is the insight that
separates someone who has thought about production LLM systems from someone who has read about
scaling web apps.

**They'll respect "I inferred this, I didn't measure it."** Overclaiming measurement is the fastest
way to lose credibility, and it's trivially exposed by "what was your p99?"

**They'll also want to hear that the current design was correct.** Building Redis fan-out and a job
queue for a single-instance beta would have been waste. Knowing when a simple design becomes wrong is
more valuable than always building for scale.

---

## 13. Common Mistakes

- **"Just add more servers."** Doesn't work for stateful connections without a shared registry, and
  does nothing about a vendor rate limit.
- **Claiming load-test numbers you don't have.** One follow-up destroys it.
- **Missing the pool insight.** It's counterintuitive and it's the most impressive thing you can say
  here.
- **Not connecting token optimization to scale.** It's your strongest contribution and it belongs in
  this conversation.
- **Answering about a vector DB you don't have.** Correct the premise politely.
- **Criticizing the current design as naive.** It was appropriate for its stage. Say when it stops
  being appropriate.

---

## 14. Cheat Sheet

**Breaks first, in order:**
1. **In-process `ConnectionManager`** (`Dict[UUID, Set[ConnectionState]]`) → no cross-instance
   delivery. Fix: Redis pub/sub per project + sticky sessions.
2. **Synchronous backtests** → pins the process. Fix: SQS + ECS/Batch workers, notify over WS.
3. **DB pool** → each turn holds a session 30–60s. 20 conns ≈ 20 concurrent turns. Fix: release the
   session across the LLM call, not a bigger pool.
4. **OpenAI TPM + cost** → the ceiling you cannot engineer around.

**Scales fine:** S3 · stateless REST · Postgres reads (replicas) · agent CPU (all I/O wait).

**Mitigations already in place:** navigator (−96% prompt) · prompt caching (static prefix) ·
compaction (20 non-user items) · model per task · per-user rate limits + monthly token quota ·
`max_turns=10`.

**Not built:** semantic cache · request queue with fair scheduling · multi-provider · CDN ·
read replicas · multi-region.

**Framing:** token optimization *is* a scalability contribution — fewer tokens per request means more
concurrent requests under a fixed TPM ceiling.

**Honesty line:** "This is structural analysis from the code, not load-test data — I didn't benchmark
it."
