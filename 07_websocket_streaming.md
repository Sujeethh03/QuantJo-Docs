# Real-Time Agent Streaming over WebSockets

**Your third headline contribution.**

---

## 1. Interview Answer (2–3 minutes)

> An agent turn on our platform takes thirty to sixty seconds. The agent reads the conversation,
> maybe resolves an asset symbol, delegates to a code specialist, which reads SDK documentation pages,
> writes a strategy file, and validates it. With a normal request/response API that's a minute of
> spinner, and users assume it's broken and refresh — which is the worst outcome, because the work is
> still running server-side and they've now queued a second turn.
>
> So the chat runs over a WebSocket, and I built the streaming path that surfaces what the agent is
> actually doing while it does it. The OpenAI Agents SDK exposes `Runner.run_streamed`, which returns
> an async iterator of run items rather than a final result. I consume that stream and dispatch on
> event type: when a tool is called, I look up a human-readable label and push a `status` event. When
> a tool returns, I inspect the output — success flips the status back to "Thinking", an error flips it
> to "Checking what went wrong". And reasoning items emit a heartbeat, which matters because the code
> specialist can reason for twenty seconds with no tool activity and the UI would otherwise look
> frozen.
>
> The important design decision is that these are *ephemeral* status events, not chat messages.
> There's a note in the codebase about this — an earlier version emitted tool progress as messages,
> which polluted the persisted conversation history with "Reading SDK reference" noise that then got
> re-sent to the model on every subsequent turn. Separating transient UI state from durable
> conversation was the fix.
>
> The other detail I'd call out is the label map. Unknown tool names emit *nothing* rather than
> falling back to the raw identifier — so if someone adds a tool without adding a label, the user sees
> the previous status persist instead of a raw string like `resolve_assets_v2` flashing on screen.
> Fail-quiet rather than fail-ugly.

---

## 2. What Problem Were We Solving?

**Business.** Perceived performance is performance. A minute of undifferentiated spinner reads as
broken; a minute of "Reading SDK reference → Generating code → Editing strategy" reads as *working*.
Users abandon the first and wait through the second. The refresh problem is worse than cosmetic —
a refresh mid-turn means a duplicate message and a doubled token bill.

**Technical.**
- Agent turns are long and multi-phase.
- The phases are known server-side but invisible client-side.
- Progress is **server-initiated** — the client cannot know when a tool starts.
- The conversation is bidirectional and stateful: accept/reject changeset events, queued messages,
  cancellation.

**Why HTTP wasn't enough:**

| Approach | Why it fails here |
|---|---|
| Request/response | No progress until the end. The default, and the problem. |
| Polling | Latency/load tradeoff with no good answer; needs server-side progress state; wasteful when idle. |
| Long polling | Better, but reconnect-per-update and awkward for bidirectional flow. |
| SSE | Genuinely viable for server→client streaming, and simpler. **But** it's unidirectional — accept/reject changeset, cancel, and queued messages all need client→server on the same channel. You'd end up running SSE plus a REST side-channel and correlating them. |
| **WebSocket** | Full-duplex, one connection, one auth handshake, natural fit for a stateful conversation. |

**Be ready for "why not SSE?"** — it's the sharpest question in this area. The answer is
bidirectionality, and you should name the specific client→server events that force it.

---

## 3. Architecture

```
BROWSER                                    FASTAPI                            OPENAI
   │                                          │                                  │
   │──── WS /ws/projects/{id} ───────────────►│                                  │
   │──── {"type":"auth","token":jwt} ────────►│ AuthService verifies JWT          │
   │                                          │ ProjectService checks ownership   │
   │◄──── {"type":"connected", ...} ──────────│                                  │
   │◄──── {"type":"agent_actions", ...} ──────│ chip strip                       │
   │                                          │                                  │
   │──── {"type":"user_message",              │                                  │
   │      "content":"add a trailing stop",    │                                  │
   │      "message_id":uuid,                  │                                  │
   │      "active_view":{...}} ──────────────►│                                  │
   │                                          │                                  │
   │                                   asyncio.create_task(_process_and_emit)    │
   │                                     ├─ own AsyncSessionLocal()              │
   │                                     ├─ own AgentContext                     │
   │                                     └─ cancel_event wired in                │
   │                                          │                                  │
   │◄──── status "Thinking" ──────────────────│                                  │
   │                                          │── Runner.run_streamed() ────────►│
   │                                          │                                  │
   │                                          │◄── reasoning_item_created ───────│
   │◄──── status "Thinking" ──────────────────│                                  │
   │                                          │◄── tool_called: propose_code_change
   │◄──── status "Editing strategy" ──────────│                                  │
   │                                          │      (Codex sub-run begins)      │
   │                                          │◄── tool_called: read_sdk_page ───│
   │◄──── status "Reading SDK reference" ─────│                                  │
   │                                          │◄── tool_called: propose_files ───│
   │◄──── status "Generating code" ───────────│                                  │
   │                                          │◄── tool_output {success:true} ───│
   │◄──── status "Thinking" ──────────────────│                                  │
   │                                          │                                  │
   │◄──── artifact_update {action:"generated",│                                  │
   │        artifacts:[{file_path,            │                                  │
   │        old_content,new_content}],        │                                  │
   │        changeset_id} ────────────────────│                                  │
   │◄──── message {content, message_id, view}─│                                  │
   │◄──── agent_actions (refreshed chips) ────│                                  │
   │◄──── project_state_changed ──────────────│                                  │
   │◄──── status null  ◄── TURN COMPLETE ─────│                                  │
   │                                          │                                  │
   │──── {"type":"user_event",                │                                  │
   │      "event":"accept_changeset",         │                                  │
   │      "changeset_id":"..."} ─────────────►│ pending/ → head/ in S3           │
```

### Event dispatch — the core of what I built

```
Runner.run_streamed(...)  →  async for event in result.stream_events()
                                      │
                          event.type == "run_item_stream_event"
                                      │
                          _handle_stream_event(event, emitter, project, tool_extras)
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
  "tool_called"               "tool_output"          "reasoning_item_created"
  name = item.raw_item.name   parse item.output      emit_status("Thinking")
  label = TOOL_LABELS.get()     ├─ "view" key  →       (heartbeat during long
  if label: emit_status(label)  │   tool_extras         silent reasoning)
  if not: EMIT NOTHING          ├─ error → "Checking
   (prev status persists)       │   what went wrong"
                                └─ success → "Thinking"
```

---

## 4. Deep Technical Explanation

### `Runner.run_streamed` vs `Runner.run`

`Runner.run()` returns the final result — you learn nothing until it's over. `Runner.run_streamed()`
returns a `RunResultStreaming` whose `.stream_events()` is an async generator yielding items as the
agent produces them.

The loop filters to `run_item_stream_event` — semantic, item-level events (a tool was called, a tool
returned, a reasoning item was created) rather than raw token deltas. That's the right granularity
here: we're showing *phase*, not typing out text character by character.

One consequence worth knowing: `RunResultStreaming` has **no `.usage`**. Token accounting comes from
`result.context_wrapper.usage`, which the SDK accumulates across tool calls — including the nested
Codex sub-run inside `propose_code_change`. There's a comment in `agent_loop.py` about exactly this.
It's the kind of detail that proves you worked on the code rather than read about it.

### The label map — and why unknown tools emit nothing

```python
TOOL_LABELS = {
    "resolve_assets":       "Resolving assets",
    "get_backtest_results": "Analysing results",
    "invoke_action":        "Opening view",
    "propose_code_change":  "Editing strategy",
    # Codex-internal tools, visible during a propose_code_change run
    "propose_files":        "Generating code",
    "read_head_file":       "Reading strategy",
    "read_sdk_page":        "Reading SDK reference",
}
```

Two things to say about this:

**Internal tool names never reach users.** `read_sdk_page` is an implementation detail; "Reading SDK
reference" is a user-facing concept. This is the same discipline as not leaking stack traces.

**Unknown tool → no event at all.** The comment in the source states it: *"Unknown tool name → no
event, so the prior label stays on screen instead of flashing a raw identifier."* If someone adds a
tool and forgets a label, the previous status persists — degraded but coherent. Falling back to the
raw name would put `resolve_assets_v2` in front of a non-technical trader. **Fail quiet, not ugly.**

Note also that the map includes **Codex-internal tools**. The sub-agent runs inside a tool call on the
parent agent, and its tool activity surfaces in the same stream. That's why the user sees "Reading SDK
reference" and "Generating code" during what is, from the top-level agent's perspective, a single
`propose_code_change` call. Nested visibility is a genuinely nice property and worth pointing out.

### Status events are ephemeral, not messages

This is the most important design decision in the feature, and it came from a real bug. From the
project changelog:

> **Tool progress cleanup**: Replaced `emit_message()` with `emit_status()` for ephemeral tool
> progress (no longer pollutes chat history).

The original implementation emitted progress as chat messages. Consequences:
1. Persisted conversation history filled with "Reading SDK reference" noise.
2. That noise was re-sent to the model on every subsequent turn — paying tokens for it repeatedly.
3. It hit the compaction threshold faster (20 non-user items), triggering summarization sooner.
4. Reloading a project showed progress spam interleaved with real conversation.

`emit_status(label)` sends a transient event with a `label` that the frontend renders as a single
replaceable indicator, and `emit_status(None)` clears it. Nothing persists.

**The general principle — separate transient UI state from durable conversation state — is what makes
this a good interview answer.** It generalizes far beyond this codebase.

### The reasoning heartbeat

```python
elif name == "reasoning_item_created":
    await emitter.emit_status("Thinking")
```

`code_expert` runs `gpt-5.3-codex` with `Reasoning(effort="medium")`. It can reason for a long stretch
with zero tool calls. Without this branch the last status would sit frozen and users would think it
hung. It's cheap, and it converts dead air into a liveness signal.

### Capturing view directives from the stream

```python
if isinstance(output, dict):
    if "view" in output:
        tool_extras["view"] = output["view"]
```

Tools can return a `view` field — a directive telling the frontend to open or focus a tab (backtest
result, optimization result, code view). The stream handler harvests it into a mutable dict that
outlives the loop, and it's attached to the final response.

*Why here?* Because the tool result is where the directive originates, and the alternative — a
separate channel for view directives — would need correlation logic. Piggybacking on a stream you're
already consuming is the simpler design. It's a small example of using an existing channel rather
than adding one.

### Turn-complete signalling

The protocol has no explicit "done" frame. Turn completion is signalled by `status` with
`label: null`. The test harness's `_await_turn_complete()` treats `status(null)`,
`artifact_update(action="generated")`, or `error` as terminal, and ignores
`message_saved`, `agent_actions`, `message_queued`, `project_state_changed`, and `connected`.

Be ready to critique this: **an implicit terminal signal is fragile.** An explicit
`{"type":"turn_complete","message_id":...}` frame would be more robust, correlate to the request, and
remove the need for a `_MAX_FRAMES=500` safety bound in the client. Say this before they do.

### Connection management and concurrency

`ConnectionManager` holds `Dict[UUID, Set[ConnectionState]]` — project → connections, so a project
open in two tabs gets both. `broadcast_to_project` deliberately **copies the connection list inside
the lock and sends outside it**, so a slow client can't block the manager. That's a real
concurrency-correctness detail worth naming.

**Message queuing:** single-slot buffer, latest-wins. If a turn is in flight, a new message replaces
whatever was pending and emits `message_queued`. *Why latest-wins rather than a FIFO queue?* Because
in a chat UI, if a user sends three messages while the agent is busy, they want a response to the
last one — the earlier ones are usually corrections. A queue would make the agent respond to stale
intent.

**Cancellation:** `ConnectionState.cancel_event` is a `threading.Event` threaded into `AgentContext`,
so tools can check it mid-execution.

### Per-turn database session — the bug that shaped the design

The docstring on `_process_and_emit` is unusually explicit, and it's the best thing in this file to
bring up unprompted:

> Opens its own `AsyncSession` for the duration of the turn — does NOT share the WebSocket-level
> session. `AsyncSession` is not safe for concurrent coroutine use (foreground message-receive loop +
> background agent task on the same session corrupts the identity map and produces sporadic
> "Instance is not persistent" errors). Per-turn isolation eliminates that race.

The agent turn runs as a background `asyncio.Task` while the receive loop keeps reading frames. Both
touching one `AsyncSession` concurrently corrupts SQLAlchemy's identity map, producing intermittent
errors that look random. The fix — a session per turn — is the correct one: **a session is a unit of
work, and concurrent units of work need separate sessions.**

There's a related detail elsewhere in `agent_loop.py`: `await db.get(Project, project.id)` is used
instead of `db.refresh(project)`, because `refresh` raises if the instance was detached by a rollback
inside a tool, while `get` is identity-map-aware. Two commits in the history (`7019dd3`
"per-turn DB session for the agent loop") back this up. **This is your best "hard bug I debugged"
story — see [14](14_my_contributions.md).**

---

## 5. My Actual Contribution

**What existed** — the WebSocket transport itself: `ConnectionManager`, `ConnectionState`, the route,
JWT-over-WS auth, and `EventEmitter` with `connected`, `message`, `artifact_update`, `error`.

**What I built / changed** — the streaming *progress* path:
- `_handle_stream_event()` in `agent_loop.py` — the dispatcher over `run_item_stream_event`.
- `TOOL_LABELS` — the user-facing label map, including Codex-internal tools, with the
  unknown-tool-emits-nothing rule.
- The `emit_message` → `emit_status` migration for tool progress, and `emit_status(None)` as the
  idle/turn-complete signal.
- The `reasoning_item_created` heartbeat.
- `view` directive harvesting from tool outputs via `tool_extras`.
- Consumption of this protocol in the test harness's `AgentWebSocketClient` — which is what forced
  the turn-complete semantics to be pinned down precisely.

**What changed** — the UI went from an opaque 30–60s spinner to a live phase indicator, and the
persisted conversation stopped accumulating progress noise.

**Result** — visible progress on every tool call, no dead air during long reasoning, no history
pollution, and a protocol precise enough that an automated client can drive it.

---

## 6. Files and Code

**Backend**

| Path | Contents |
|---|---|
| `quantjo_api/routes/websocket.py` | WS endpoint, auth handshake, receive loop, `_process_and_emit`, per-turn session, accept/reject routing |
| `quantjo_api/agent/connection_manager.py` | `ConnectionState`, `ConnectionManager`, `connection_manager` singleton, queuing, cancellation |
| `quantjo_api/agent/event_emitter.py` | `EventEmitter` — `emit_status`, `emit_message`, `emit_changeset_update`, `emit_agent_actions`, `emit_project_state_changed`, `emit_message_saved`, `emit_message_queued`, `emit_error`, `emit_token_usage`, `emit_connected` |
| `quantjo_api/agent/agent_loop.py` | `TOOL_LABELS`, `_handle_stream_event`, `Runner.run_streamed` consumption, `tool_extras` |
| `quantjo_api/agent/context.py` | `AgentContext` — carries `emitter` and `cancel_event` to tools |
| `quantjo_api/agent/services/changeset_handler.py` | Accept/reject handling |

**Test harness**

| Path | Contents |
|---|---|
| `client/websocket_client.py` | `AgentWebSocketClient`, `_await_turn_complete`, `_MAX_FRAMES=500`, `_IGNORED_TYPES` |
| `client/protocol.py` | `AgentClient`, `TurnResult`, `ToolCallEvent` |

**Server → client events:** `connected`, `status`, `message`, `message_saved`, `artifact_update`,
`agent_actions`, `project_state_changed`, `message_queued`, `token_usage`, `error`.

**Client → server:** `auth`, `user_message`, `user_event` (`accept_changeset` / `reject_changeset`).

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| WebSocket over SSE | Bidirectional: accept/reject, cancel, queued messages on one channel | Stateful connections complicate horizontal scaling |
| Item-level events, not token deltas | We show *phase*, not typing | No token-by-token text streaming |
| Status ephemeral, not messages | Keeps history clean and cheap | Progress isn't recoverable after reload |
| Unknown tool → no event | Never show raw identifiers to traders | New tools silently lack labels |
| Reasoning heartbeat | Long silent reasoning looks like a hang | Slightly noisy status churn |
| `status(null)` as turn-complete | Reuses an existing event | Implicit; fragile; needs `_MAX_FRAMES` guard |
| Latest-wins single buffer | Users want the last message answered | Earlier messages silently dropped |
| Per-turn `AsyncSession` | `AsyncSession` isn't coroutine-safe | Extra connection per turn |
| Broadcast copies list inside lock, sends outside | Slow client can't block the manager | Brief window where a stale connection gets a send |
| JWT in first frame, not query string | Keeps tokens out of URLs and logs | Extra handshake round trip |

---

## 8. Production Improvements

1. **Explicit `turn_complete` frame** with `message_id` correlation. Removes the implicit signal and
   the frame-count guard.
2. **Redis pub/sub for fan-out.** `ConnectionManager` is in-process — two API instances can't reach
   each other's connections. This is *the* thing that breaks first under horizontal scaling. See
   [11](11_scalability.md).
3. **Heartbeat/ping-pong and reconnect with resume.** The test client sets `ping_interval=20`; the
   server should enforce liveness too, and a `Last-Event-ID`-style resume would let a dropped client
   recover mid-turn instead of losing the response.
4. **Backpressure.** `send_json` catches exceptions and prints. A slow consumer should be detected
   and disconnected, not silently accumulate.
5. **Structured logging instead of `print`.** `connection_manager.py` uses `print()` for lifecycle
   events — fine in dev, wrong in production.
6. **Token-level streaming** for the final message so text appears as it's generated.
7. **Per-connection rate limiting** at the transport layer, not just per-user in the agent loop.
8. **Protocol versioning** — a `version` field in the handshake so client and server can evolve
   independently.
9. **Metrics** — turn duration histograms, per-tool latency, disconnect rate, queue depth.

---

## 9. Common Interview Questions

1. Why WebSockets and not HTTP?
2. Why not Server-Sent Events?
3. Why not polling?
4. Walk me through a full turn's event sequence.
5. How does authentication work over a WebSocket?
6. How do you know a turn is finished?
7. What's the difference between a `status` event and a `message` event?
8. Why don't tool-progress updates go into chat history?
9. What happens if a user sends two messages quickly?
10. How does cancellation work?
11. What happens if the connection drops mid-turn?
12. Why does each turn get its own database session?
13. What happens on an unknown tool name?
14. How does a nested sub-agent's tool activity reach the client?
15. How do you scale WebSockets horizontally?
16. What's a view directive and how does it travel?
17. How does the test harness drive this protocol?
18. What's the failure mode if the frontend ignores `status(null)`?
19. How do you prevent one slow client from affecting others?
20. Would you design the protocol the same way again?

---

## 10. Strong Answers

**Q2 — Why not SSE?**
> SSE was genuinely the closer call, and for pure server-push it's simpler — it's plain HTTP, it
> reconnects automatically, and it works through proxies that mangle WebSocket upgrades. The reason
> it doesn't fit is that our channel isn't one-directional. The client sends the user message, sends
> accept and reject events for proposed changesets, and can cancel a running turn. With SSE I'd need
> an SSE stream for downstream plus REST calls for upstream, and then correlation logic to tie a POST
> to accept a changeset back to the right stream. That's two transports and a correlation problem
> instead of one connection. If the product had been read-only streaming — say, live backtest progress
> with no user interaction — I'd have picked SSE without hesitating.

**Q8 — Why not in chat history?**
> That was a real bug we fixed. The first version emitted tool progress with `emit_message`, so
> "Reading SDK reference" was persisted as a chat message. Three things went wrong. The stored
> conversation filled with noise, so reloading a project showed progress spam interleaved with real
> messages. That noise was re-sent to the model on every subsequent turn, so we paid input tokens for
> it repeatedly. And it counted toward our compaction threshold — twenty non-user items — so
> conversations got summarized much sooner than they should have. The fix was `emit_status`, which
> sends a transient event the frontend renders as a single replaceable indicator and clears with
> `emit_status(None)`. The general principle is that transient UI state and durable conversation state
> are different things and shouldn't share a channel.

**Q12 — Per-turn DB session.**
> This was the nastiest bug I dealt with. The agent turn runs as a background asyncio task so the
> receive loop can keep reading frames — that's what makes cancellation and message queuing possible.
> Originally both shared the WebSocket-level `AsyncSession`. `AsyncSession` isn't safe for concurrent
> coroutine use: two coroutines touching it interleave and corrupt SQLAlchemy's identity map, and what
> you see is sporadic "Instance is not persistent" errors that don't reproduce reliably because they
> depend on timing. The fix was giving each turn its own `AsyncSessionLocal()` and building a fresh
> `AgentContext` bound to it, so tools within a turn share one transaction and separate turns share
> nothing. There's a related detail in the agent loop — we use `db.get(Project, id)` rather than
> `db.refresh(project)`, because `refresh` raises if a tool rolled back and detached the instance,
> whereas `get` is identity-map-aware and just returns it.

**Q15 — Horizontal scaling.**
> It doesn't scale horizontally as built, and that's the first thing I'd fix. `ConnectionManager`
> keeps connections in an in-process dictionary, so if a user connects to instance A and something
> needs to notify them from instance B, instance B can't reach them. For the chat path specifically
> that's tolerable, because the turn runs on the same instance holding the socket — but anything
> server-initiated from elsewhere, like a paper-trade signal or a backtest completing on a worker,
> breaks. The fix is standard: Redis pub/sub with a channel per project, each instance subscribing for
> its local connections, plus sticky sessions or a connection-routing layer at the load balancer. I'd
> also want the connection registry externalized so you can answer "is this user connected anywhere"
> without asking every instance.

**Q20 — Design it the same way?**
> Mostly, with one clear change: I'd add an explicit `turn_complete` frame carrying the `message_id`.
> Right now turn completion is signalled by a `status` event with a null label, which works but is
> implicit and order-dependent — the test client has to treat three different event types as terminal
> and carries a 500-frame safety bound in case none arrives. An explicit terminal frame correlated to
> the request would make the protocol self-describing, let the client match responses to requests, and
> remove that guard entirely. I'd also version the protocol in the handshake, because once you have an
> automated client consuming it, changing event semantics silently breaks things.

---

## 11. Follow-up Questions

- "Your `send_json` catches every exception and prints. What happens to a half-dead connection?"
- "If the browser reloads mid-turn, is the response lost?"
- "You broadcast to all connections for a project. Is that an authorization concern with sharing?"
- "`cancel_event` is a `threading.Event` in an async codebase. Why?"
- "What's the memory cost of an idle connection, and what's your ceiling per instance?"
- "How would you add token-level text streaming without breaking the current protocol?"
- "Latest-wins drops messages. Has that ever surprised a user?"

---

## 12. Senior Engineer Perspective

**The transient-vs-durable insight is the strongest thing here.** Recognizing that progress updates
and conversation messages are different kinds of state — with different lifetimes, storage, and cost
implications — generalizes to a lot of systems. Lead with it.

**They will go straight to horizontal scaling.** The in-process `ConnectionManager` is an obvious
single-instance assumption. Naming it before they do, with the Redis pub/sub fix, turns a weakness
into evidence of judgment.

**They will appreciate the concurrency bug.** The `AsyncSession` identity-map corruption is a real,
subtle, hard-to-reproduce bug with a principled fix. It's the best technical story in your internship
— it shows debugging under uncertainty and understanding of a framework's threading model.

**They may probe the protocol design.** Implicit turn-completion is a legitimate weakness. Have the
critique ready.

**They will ask about backpressure and failure.** Currently `send_json` swallows exceptions. Know
that, and know what production would do instead.

---

## 13. Common Mistakes

- **"WebSockets are faster than HTTP."** They aren't inherently. The reason is bidirectionality and
  server push, not speed.
- **Not having an SSE answer.** It's the obvious alternative and the question separates people who
  chose from people who defaulted.
- **Claiming the whole WebSocket layer.** The transport existed. You built the *streaming progress
  path* on top of it. That's a clean, defensible, still-impressive claim.
- **Ignoring scaling.** Every interviewer asks. "In-process registry, needs Redis pub/sub" is the
  answer.
- **Missing the history-pollution story.** It's the most interesting design decision and it came from
  a real bug — those are the stories interviewers remember.
- **Saying "it streams the agent's thoughts."** It streams *item-level events*: tool called, tool
  returned, reasoning item created. Precision matters.

---

## 14. Cheat Sheet

- **Endpoint:** `WS /ws/projects/{id}`. Auth = JWT in the first frame (`{"type":"auth","token":...}`),
  not the query string.
- **Why WS not SSE:** bidirectional — `accept_changeset`, `reject_changeset`, cancel, queued messages.
- **Mechanism:** `Runner.run_streamed()` → `async for event in result.stream_events()` → filter
  `run_item_stream_event` → `_handle_stream_event`.
- **Three branches:** `tool_called` → `TOOL_LABELS` lookup · `tool_output` → error/success status +
  harvest `view` · `reasoning_item_created` → "Thinking" heartbeat.
- **Unknown tool → emit nothing** (prior label persists; never show raw identifiers).
- **Status ≠ message.** `emit_status` is ephemeral; `emit_message` persists. The `emit_message` →
  `emit_status` migration fixed history pollution, repeated token cost, and early compaction.
- **Turn complete:** `status` with `label: null`. (Implicit — a known weakness.)
- **Events out:** `connected`, `status`, `message`, `message_saved`, `artifact_update`,
  `agent_actions`, `project_state_changed`, `message_queued`, `token_usage`, `error`.
- **Queuing:** single-slot, latest-wins → `message_queued`. **Cancel:** `threading.Event` in
  `AgentContext`.
- **Per-turn `AsyncSession`** — `AsyncSession` is not coroutine-safe; sharing corrupts the identity
  map → "Instance is not persistent".
- **Usage:** `RunResultStreaming` has no `.usage` → read `result.context_wrapper.usage` (accumulates
  the Codex sub-run).
- **Scaling gap:** in-process `ConnectionManager` → needs Redis pub/sub + sticky sessions.
