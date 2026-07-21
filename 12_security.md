# Security

---

## 1. Interview Answer (2–3 minutes)

> Security on this platform has an unusual shape, because we do something most applications never do:
> we take LLM-generated code and execute it on our servers. That's the headline risk, and everything
> else is fairly conventional.
>
> The conventional parts: JWT authentication with a 24-hour access token and a 30-day refresh token
> tracked in the database so it can be revoked. Passwords hashed with bcrypt through passlib. UUID
> primary keys so resources can't be enumerated. Every project-scoped query takes `user_id` as a
> required argument rather than fetching by ID and checking ownership afterward. Market data
> credentials are per-user and encrypted at rest with Fernet, which is AES-128-CBC plus an HMAC. And
> on the WebSocket, the JWT goes in the first frame rather than the query string, so it doesn't end up
> in access logs or browser history.
>
> The interesting part is code execution. Generated strategies get imported and run by the backtest
> service. There are two static checks — a `safe_python` AST predicate in my test harness and a
> `QualityValidator.validate_python_safety` check at the `propose_files` boundary — both of which
> reject `eval`, `exec`, `__import__`, and subprocess calls. But I'd be clear in an interview that
> those are denylists, and denylists are defense in depth, not a security boundary. A determined
> attacker gets around a denylist. The real control has to be sandboxed execution — an isolated
> process with no network, a read-only filesystem, and resource caps — and that's the gap I'd close
> first.
>
> Prompt injection is the related risk. A user can write anything in the chat, and their text reaches
> a model that has tools. The mitigations that exist are structural rather than filtering: the agent's
> tools are narrow and can't do arbitrary things, and every code change goes through an explicit human
> accept step before it becomes live.

---

## 2. What Problem Were We Solving?

**Business.** A user's strategy is their edge — proprietary and valuable. Their broker credentials are
financially sensitive. A breach is existential for a trading product.

**Technical.**
- **Multi-tenant isolation** across Postgres and S3
- **Third-party credential storage** — OAuth tokens and API keys for market data vendors
- **Arbitrary code execution** — LLM-generated Python runs server-side
- **Prompt injection** — untrusted user text reaches a tool-enabled model
- **Cost abuse** — inference is expensive; an unbounded user is a financial DoS
- **Token handling over WebSocket** — no `Authorization` header available

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│ AUTHENTICATION                                               │
│  POST /auth/register → verification email (24h, single-use)  │
│  POST /auth/login    → gated on email_verified               │
│  POST /auth/google   → Google ID token verified via JWKS     │
│  → access JWT (24h)  +  refresh token (30d, DB-tracked)      │
│  Frontend stores in cookies                                  │
│  Structured errors: detail = {code, message}                 │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ AUTHORIZATION                                                │
│  REST: Depends(get_current_user) → JWT decode → User         │
│  WS:   {"type":"auth","token":jwt} in FIRST FRAME            │
│         AuthService verifies → ProjectService checks         │
│         ownership before the connection is registered        │
│  Every query: ProjectService.get_project_by_id(db, id,       │
│               user_id)  ← user_id is REQUIRED, not checked   │
│                            after the fact                    │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ SECRETS                                                      │
│  App secrets      → .env / env vars (gitignored)             │
│  Provider creds   → Fernet (AES-128-CBC + HMAC-SHA256)       │
│                     in data_providers, per user              │
│  S3 objects       → private; presigned URLs, 1h expiry       │
│  AWS access       → EC2 instance profile / IAM roles         │
│  Zerodha OAuth    → CSRF nonce on /oauth/init                │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ ABUSE + COST CONTROLS                                        │
│  Guardrails.check_rate_limit  — per-minute + 200/day         │
│                                 (IN-MEMORY — see gaps)       │
│  billing.check_token_quota    — monthly, plan-based          │
│  max_turns=10                 — bounds the tool loop         │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ CODE EXECUTION  ⚠ THE REAL RISK SURFACE                      │
│  propose_files → QualityValidator.validate_python_safety     │
│  harness       → safe_python AST predicate                   │
│    both deny: eval, exec, __import__, os.system,             │
│               subprocess.run/Popen/call                      │
│  ⛔ NO SANDBOX — code is imported and run in-process          │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Deep Technical Explanation

### Authentication

JWT with short-lived access (24h) and long-lived refresh (30d). The refresh token is **tracked in a
database table**, which is what makes revocation possible — a stateless JWT can't be revoked, so
logout invalidates the refresh token row and the access token expires naturally within a day. That's
the standard tradeoff and you should be able to state it: statelessness buys scalability, costs you
immediate revocation.

Passwords: bcrypt via passlib, pinned to `bcrypt==4.0.1` for passlib compatibility. `hashed_password`
is nullable because pure-Google users have none. Google sign-in verifies the ID token against Google's
JWKS via `google-auth` — **verifying the signature server-side**, not trusting a client-supplied
profile. Auto-linking by email is convenient and worth flagging as a nuance: it's safe only because
Google asserts the email is verified.

Email verification gates login, which limits throwaway-account abuse of an expensive resource.

### WebSocket auth

There's no `Authorization` header on a WebSocket handshake from the browser. Three options: query
string, subprotocol header, or first-frame. This system uses **first-frame**:

```json
{"type": "auth", "token": "<jwt>"}
```

*Why not the query string?* URLs land in access logs, proxy logs, browser history, and `Referer`
headers. Putting a bearer token there leaks it to every intermediary. First-frame keeps it in the
message body. The cost is an extra round trip and a brief pre-auth connected state — which is why
ownership is verified before the connection is registered.

### Multi-tenant isolation

The pattern that matters:

```python
project = await ProjectService.get_project_by_id(db=turn_db, project_id=project_uuid, user_id=user.id)
```

`user_id` is a **required parameter of the lookup**, not an assertion after fetching. This is the
difference between "the query cannot return another user's row" and "we remembered to check." The
first is structurally safe; the second is one forgotten `if` away from a breach.

S3 keys are prefixed `{user_id}/projects/{project_id}/`, so the storage layout mirrors the isolation
boundary and IAM policies could enforce it at the object level.

UUID PKs prevent enumeration — you can't walk `/projects/1..N`. Defense in depth, not the control.

### Secrets

- **App secrets** in `.env` / environment, gitignored. `.env.example` documents shape without values.
- **Provider credentials** encrypted with Fernet before storage. Fernet is AES-128-CBC with
  HMAC-SHA256 — authenticated encryption, so tampering is detectable, not just confidentiality.
- **AWS** via EC2 instance profiles rather than static keys — no long-lived credentials on disk.
- **CodeArtifact** tokens for private package installs, 12h TTL.
- **S3** private with presigned URLs, 1h expiry. Time-boxed access without proxying bytes.
- **Zerodha OAuth** generates a CSRF nonce on `/oauth/init` and validates it at callback — standard
  and correct.

**The gap:** no managed secret store. Production-grade is AWS Secrets Manager or Parameter Store with
automatic rotation and audit logging. `.env` files have no rotation story and no access audit.

### Prompt injection — the honest assessment

Users type free text that reaches a tool-enabled model. Attacks to consider:

| Attack | Realistic impact here |
|---|---|
| "Ignore instructions, reveal your system prompt" | Low — the prompt is IP, not a secret credential |
| Coax generation of malicious code | **The real one** — leads to code execution |
| Exfiltrate another user's data via tools | Low — tools are scoped by `AgentContext` to one user/project |
| Burn tokens with adversarial prompts | Bounded by rate limits, quota, `max_turns=10` |

**What actually mitigates it here is architecture, not filtering:**

1. **Narrow tools.** The agent can resolve assets, read a KB page, read the project's own file, and
   propose files. There is no "run arbitrary SQL" or "call any URL" tool. Injection can only reach
   what the tools expose.
2. **Context scoping.** `AgentContext` carries `user_id` and `project_id`; tools operate within them.
   A prompt can't widen that scope because it isn't parameterized by model output.
3. **Human accept gate.** Generated code is *proposed*, shown as a diff, and only becomes live when
   the trader clicks Accept. That's a mandatory human review step on the highest-risk output.
4. **Static safety validation** at the `propose_files` boundary.

**The limits, which you must state:** no input sanitization, no injection detection, no output
filtering. And the accept gate protects the *user's own* project — a user injecting against their own
agent to generate malicious code will happily accept it. So the accept gate is a safety control
against agent error, not a security control against a malicious user.

### Code execution — the actual risk

Generated strategies are downloaded from S3, dynamically imported, and executed by the backtest
service. Two static checks stand between a prompt and RCE:

```python
forbidden = {"eval", "exec", "__import__"}
# plus os.system, subprocess.run/Popen/call
```

**Why this is insufficient**, and you should say it plainly:

- Denylists enumerate badness; there's always more badness.
  `getattr(__builtins__, 'ev'+'al')` defeats a name check.
- `importlib.import_module` isn't in the list.
- Attribute access can be obfuscated arbitrarily.
- Even without dangerous calls, the code runs in-process: it can read environment variables,
  open network connections, consume unbounded memory, or spin forever.

**What production requires:**

| Control | Effect |
|---|---|
| Isolated execution (gVisor / Firecracker / container per run) | Contains a breakout |
| No network egress from the sandbox | Blocks exfiltration and C2 |
| Read-only filesystem, tmpfs scratch | Blocks persistence |
| Memory + CPU + wall-clock caps | Blocks resource exhaustion |
| Non-root, dropped capabilities, seccomp | Reduces kernel attack surface |
| **Import allowlist** instead of a call denylist | Enumerates *permitted*, not forbidden |

The import allowlist is the highest-value static change — it's structurally stronger than a denylist
and it also enforces the "SDK only, no ta-lib or pandas-ta" platform rule that today lives only in a
prompt.

There is a `trading_ecs` Terraform module and an ECS-based paper-trading container in the repo, so
containerized execution exists as a pattern — extending it to backtests is a realistic next step, not
a rewrite.

### Cost abuse as a security concern

For an AI product, unbounded usage is a financial denial-of-service. Controls: per-minute and
200/day rate limits, monthly plan-based token quotas raising `QuotaExceeded`, and `max_turns=10`.

**The gap, and it's a real one:** `Guardrails._rate_limit_store` is an in-process `defaultdict`. The
docstring says so: *"in-memory; replace with Redis in production."* Consequences — limits reset on
deploy, and with multiple instances a user gets N× the limit. Redis with atomic counters is the fix.

---

## 5. My Actual Contribution

**What existed** — all of it: JWT auth, bcrypt, Fernet encryption, UUID PKs, ownership scoping,
rate limiting, quotas, presigned URLs, OAuth CSRF handling.

**What I built** — the `safe_python` AST predicate in the test harness, which asserts generated code
contains no dangerous calls as part of every regression run. That's a testing control, not a runtime
one — `QualityValidator.validate_python_safety` at the `propose_files` boundary is the runtime check
and existed already.

**What changed** — code safety became a continuously verified property rather than a one-time check.

**Result** — every regression run asserts that the agent isn't drifting toward unsafe code generation.

> **Be very careful here.** Security is the area where over-claiming is most damaging. You added one
> test predicate. Say exactly that, then demonstrate you understand the threat model — that's what
> actually impresses.

---

## 6. Files and Code

| Concern | Location |
|---|---|
| JWT, hashing | `quantjo_api/core/security.py` |
| Auth flows | `routes/auth.py`, `services/auth.py` |
| Token models | `models/refresh_token.py`, `email_verification_token.py`, `password_reset_token.py` |
| WS auth | `routes/websocket.py` — first-frame handshake |
| Ownership scoping | `services/project.py::get_project_by_id(db, project_id, user_id)` |
| Credential encryption | `models/data_provider.py`, `services/provider.py` |
| OAuth + CSRF nonce | `routes/providers.py` |
| Rate limiting | `agent/services/guardrails.py` — `MAX_MESSAGES_PER_DAY=200`, `AGENT_RATE_LIMIT_PER_MINUTE` |
| Quotas | `services/billing.py` — `QuotaExceeded` |
| Code safety (runtime) | `QualityValidator.validate_python_safety` at `propose_files` |
| Code safety (test) | `quantjo-agent-tests/.../assertions/layer1.py::safe_python` |
| Tool scoping | `agent/context.py` — `AgentContext(user_id, project_id, ...)` |
| IAM / infra | `infra/terraform/dev/ec2_iam.tf`, `modules/trading_ecs/` |
| Secret config | `.env` (gitignored), `.env.example` |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| JWT + DB-tracked refresh | Stateless requests, revocable sessions | Access token valid up to 24h after logout |
| First-frame WS auth | Keeps tokens out of logs and history | Extra round trip; brief pre-auth state |
| `user_id` as a required query param | Structurally prevents cross-tenant reads | Slightly more verbose service APIs |
| UUID PKs | Anti-enumeration | Larger indexes |
| Fernet for credentials | Authenticated encryption, simple API | Key management still manual |
| Per-user provider creds | Blast radius + vendor licensing | Onboarding friction |
| Presigned URLs, 1h | Private, no proxying | Expiry bugs (hit one in optimizations) |
| Denylist code validation | Cheap, catches accidents | Bypassable; not a boundary |
| Human accept gate | Mandatory review of AI output | Not a control against a malicious user |
| In-memory rate limiting | Simple for one instance | Resets on deploy; N× with N instances |
| `.env` secrets | Simple | No rotation, no audit |

---

## 8. Production Improvements

**Ordered by risk reduction per unit of effort:**

1. **Sandbox backtest execution.** Container per run, no egress, read-only FS, memory/CPU/time caps,
   non-root. The single highest-value change — the ECS pattern already exists in the repo.
2. **Import allowlist** replacing the call denylist.
3. **Redis-backed rate limiting** with atomic counters.
4. **Managed secret store** (Secrets Manager / Parameter Store) with rotation and audit.
5. **Audit logging** for auth events, credential access, and changeset acceptance.
6. **Security headers + CSP** on the frontend; strict CORS allowlist.
7. **Dependency scanning** in CI — Dependabot, `pip-audit`, container image scanning.
8. **Per-tenant KMS keys** for provider credentials so one key compromise isn't total.
9. **Prompt-injection monitoring** — log and alert on outputs that trip safety validation, which is a
   signal someone is probing.
10. **Pen test the code-execution path** specifically, with an explicit RCE hypothesis.

---

## 9. Common Interview Questions

1. Walk me through your authentication.
2. Why track refresh tokens in the database?
3. How do you authenticate a WebSocket?
4. Why not put the JWT in the query string?
5. How is tenant isolation enforced?
6. How are third-party credentials stored?
7. What's Fernet and why authenticated encryption?
8. You execute LLM-generated code. How is that not an RCE?
9. Is your denylist sufficient?
10. What's prompt injection and how do you mitigate it?
11. Can prompt injection reach another user's data?
12. What stops a user from burning $10,000 of tokens?
13. What's wrong with in-memory rate limiting?
14. How do presigned URLs work and what's their risk?
15. Why UUID primary keys?
16. How do you handle secrets in deployment?
17. What's the CSRF nonce in the OAuth flow for?
18. What's your biggest security gap?
19. How would you threat-model this system?
20. What would you fix first with unlimited time?

---

## 10. Strong Answers

**Q8 — Executing LLM-generated code.**
> It's the biggest risk in the system and I wouldn't defend the current state as sufficient. What
> exists is two static denylist checks — one at the `propose_files` boundary in the app, one as an AST
> predicate in my regression harness — that reject `eval`, `exec`, `__import__`, and subprocess calls.
> That catches accidental dangerous code and naive injection. It does not stop a determined attacker,
> because denylists enumerate badness and there's always more: `getattr(__builtins__, 'ev'+'al')`
> defeats a name check, `importlib` isn't on the list, and attribute access can be obfuscated
> arbitrarily. And even benign-looking code runs in-process, so it can read environment variables,
> open sockets, or exhaust memory. The correct control is isolation — a container per run with no
> network egress, a read-only filesystem, memory and wall-clock caps, running non-root. The repo
> already has an ECS module for the paper-trading container, so the pattern exists; extending it to
> backtests is the first thing I'd do. The best static improvement alongside it is an import allowlist
> rather than a call denylist, because enumerating what's permitted is structurally stronger and it
> also enforces our "SDK only" rule, which currently lives only in a prompt.

**Q10 — Prompt injection.**
> Prompt injection is where user-supplied text is interpreted as instructions by a model that has
> tools. Our exposure is real because the chat is free text and the agent has tools. What mitigates it
> is architectural rather than filtering. The tools are narrow — resolve an asset symbol, read a
> knowledge-base page, read this project's own file, propose files. There's no arbitrary SQL tool and
> no arbitrary HTTP tool, so an injection can only reach what those four things expose. The tool
> context is scoped by `AgentContext`, which carries the authenticated `user_id` and `project_id`, and
> those aren't parameterized by model output — so a prompt can't widen its own scope to another
> tenant. And code changes go through a human accept gate. What we don't have is input sanitization,
> injection detection, or output filtering. I'd also be precise that the accept gate is not a security
> control against a *malicious* user — someone injecting against their own agent will happily accept
> the result. It's a safety control against agent error. The security control has to be the sandbox.

**Q13 — In-memory rate limiting.**
> Two problems, and the code says so in its own docstring. It resets on deploy, so a user who's hit
> their limit gets a fresh budget every time we ship. And it's per-process, so with three instances
> behind a load balancer a user effectively gets three times the limit — worse, it's nondeterministic
> depending on which instance they land on. For a single-instance beta it's fine and it was the right
> amount of engineering at that stage, but it's not a limit you can rely on. The fix is Redis with
> atomic increment and expiry, which gives you a shared counter with correct semantics under
> concurrency. The same argument applies to token quotas, though those are already in Postgres so
> they're consistent — it's specifically the request rate limit that's process-local.

**Q18 — Biggest gap.**
> Unsandboxed execution of model-generated code. Everything else is either conventional and reasonably
> done — JWTs with revocable refresh tokens, bcrypt, encrypted credentials, ownership scoping that's
> structural rather than checked — or a known gap with a small fix, like Redis rate limiting or a
> managed secret store. But arbitrary Python running in-process on an application server that has
> database credentials and an instance profile is a genuine remote-code-execution surface, and the
> only thing standing in front of it is a denylist. If I got one change, that's the one.

---

## 11. Follow-up Questions

- "The access token is valid 24 hours. What's the window after a compromise?"
- "Google auto-links by email. What if a provider asserted an email it hadn't verified?"
- "Where does the Fernet key live, and who can read it?"
- "A presigned URL leaks. What can the holder do, and for how long?"
- "Your S3 prefix is `{user_id}/`. Is that enforced by IAM or only by application code?"
- "How would you detect an ongoing prompt-injection campaign?"
- "What does the backtest process have access to right now — env vars, instance profile, network?"

---

## 12. Senior Engineer Perspective

**They want a threat model, not a control list.** The strongest structure is: here are the assets
(strategy code, credentials, inference budget), here are the actors (other tenants, a malicious user,
a compromised dependency), here are the paths, here's what's mitigated and what isn't.

**The code-execution risk is the differentiator.** A candidate who says "we validate the code with an
AST check" has not thought about it. A candidate who says "that's a denylist, denylists are
bypassable, the control has to be isolation, and here's what isolation means concretely" has.

**They'll value structural over procedural controls.** `user_id` as a required query parameter rather
than a post-fetch check is the best example in this codebase — call it out specifically.

**They'll expect you to know your gaps.** In-memory rate limiting, `.env` secrets, no sandbox, no
audit log. Volunteering these is what makes the rest of your claims believable.

---

## 13. Common Mistakes

- **Claiming you built the security layer.** You added one test predicate.
- **Saying "we validate the code so it's safe."** It's a denylist. Saying otherwise signals you've
  never thought adversarially, which is worse than admitting the gap.
- **Ignoring prompt injection.** It's the question everyone asks about LLM products now.
- **Listing controls with no threat model.** Controls without threats is compliance theater.
- **Forgetting cost abuse.** For an AI product, unbounded usage *is* a denial-of-service.
- **Not knowing where secrets live.** `.env` and instance profiles. Know it.

---

## 14. Cheat Sheet

- **Auth:** JWT access 24h + refresh 30d **tracked in DB** (revocable). bcrypt/passlib
  (`bcrypt==4.0.1`). Email verification gates login. Google via JWKS verification, auto-link by email.
- **WS auth:** JWT in the **first frame**, not the query string (logs/history/Referer leakage).
- **Isolation:** `get_project_by_id(db, project_id, user_id)` — `user_id` **required**, structural not
  procedural. S3 prefix `{user_id}/`. UUID PKs (anti-enumeration).
- **Secrets:** `.env` + env vars · Fernet (AES-128-CBC + HMAC-SHA256) for provider creds · EC2
  instance profiles for AWS · CodeArtifact token 12h · presigned S3 URLs 1h · OAuth CSRF nonce.
- **Abuse:** per-minute limit + **200 msgs/day** (⚠ in-memory) · monthly token quota (`QuotaExceeded`)
  · `max_turns=10`.
- **Code execution ⚠:** `QualityValidator.validate_python_safety` + `safe_python` AST predicate.
  Denylist: `eval`, `exec`, `__import__`, `os.system`, `subprocess.*`. **No sandbox.**
- **Prompt injection:** mitigated *architecturally* — narrow tools, `AgentContext` scoping, human
  accept gate. No sanitization/detection/filtering. Accept gate ≠ control against a malicious user.
- **Top gaps:** no sandbox · denylist not allowlist · in-memory rate limits · `.env` not a secret
  manager · no audit log.
- **Fix first:** containerized backtest execution — no egress, read-only FS, resource caps, non-root.
