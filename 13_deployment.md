# Development, Testing, Deployment, Monitoring

> **Honesty note:** this is the weakest area of the internship in terms of your personal ownership.
> There are no CI workflow files in the repos — `.github/` contains only `CODEOWNERS`, a PR template,
> and contributing docs. Infrastructure is Terraform, and deployment is EC2-based with a documented
> setup runbook. Know it accurately; don't invent a pipeline.

---

## 1. Interview Answer (2–3 minutes)

> Development was local-first. The backend runs under `uv` — `uv run uvicorn quantjo_api.main:app
> --port 8000 --reload` — against a local PostgreSQL database with Alembic migrations and a seed
> script. The frontend is `npm run dev` on port 3000. Dependencies are managed with `uv` and locked;
> the SDK itself comes from a private AWS CodeArtifact repository, which means you need a
> twelve-hour auth token to install, and there's a setup script that refreshes it.
>
> Testing is where I made the biggest contribution, and also where I'd be honest about the state of
> things. The API layer had no automated tests — the repo said so explicitly, "Manual Testing Only,"
> with Postman collections used for manual verification. What I built was the agent regression
> harness, which is a separate package with its own CLI and a pytest plugin, running twenty multi-turn
> conversations against a live server. It's runnable in CI but I didn't wire it into a deploy gate —
> that was the next step.
>
> Infrastructure is Terraform, with a bootstrap stack for remote state, a dev environment, and a
> module for the ECS setup used by the paper-trading container. The API itself runs on EC2 with an
> instance profile for AWS access, and there are documented runbooks for EC2 setup and installation.
> HTTPS is required, because Zerodha's OAuth callback demands it.
>
> Monitoring is the thinnest part. There's structured request logging through a `DebugLogger` with
> phase timing, and token usage per turn is persisted to the database, which is genuinely useful for
> cost attribution. But there's no APM, no distributed tracing, no alerting, and some lifecycle
> logging still uses `print`. If I were taking this to production, observability is where I'd start.

---

## 2. What Problem Were We Solving?

**Business.** A small team shipping a public beta needs to move fast without breaking a product where
failures cost users money. Reproducible environments and a quality gate on the AI layer matter more
than pipeline sophistication at that stage.

**Technical.**
- A **private SDK** distributed through CodeArtifact, requiring authenticated installs
- **Two repos** (API, UI) plus a test harness that must be version-compatible
- **Database migrations** against a live beta
- **HTTPS mandatory** for OAuth callbacks
- **An AI layer whose quality can't be verified by conventional tests**

---

## 3. Architecture

```
DEVELOPMENT
  ./scripts/setup.sh            → CodeArtifact token (12h) + uv sync
  createdb quantjo_dev
  uv run alembic upgrade head
  uv run python scripts/seed_dev_data.py
  uv run uvicorn quantjo_api.main:app --port 8000 --reload
  npm run dev                   → :3000
  seeded dev login (credentials in scripts/seed_dev_data.py — not reproduced here)

TESTING
  ┌────────────────────────────────────────────────────────┐
  │ API layer      → Postman collections (MANUAL)          │
  │                  postman/collections/, environments/   │
  │ Agent layer    → quantjo-agent-tests (AUTOMATED)       │
  │                    quantjo-agent-test run              │
  │                    or pytest via the pytest11 plugin   │
  │                    → reports/{ts}.json + .md           │
  │                    → baselines/last_green.json          │
  │ ⛔ no CI workflow files in either repo                  │
  └────────────────────────────────────────────────────────┘

INFRASTRUCTURE (Terraform)
  infra/terraform/
    bootstrap/        remote state
    dev/              main.tf, ec2_iam.tf, backend.tf, variables.tf
    modules/
      trading_ecs/    ECS task/service for the paper-trading container

DEPLOYMENT
  EC2 (ap-south-1)  ── instance profile → S3, CodeArtifact read
  PostgreSQL 17
  HTTPS via Let's Encrypt (required by Zerodha OAuth)
  scripts/push_trading_image.sh → ECR → ECS (trading container)
  docs/EC2_SETUP.md, docs/INSTALLATION.md

MONITORING
  DebugLogger — request start/end, phase timing, guardrail checks,
                agent response length
  token_usage table — prompt/completion/total/cached per message,
                      per user, per project, per model
  scripts/usage_report.py — beta usage reporting
  ⛔ no APM · no distributed tracing · no alerting · some print()
```

---

## 4. Deep Technical Explanation

### `uv` and the private SDK

`uv` is used over pip/poetry for resolution speed and its lockfile. The complication is
`quantjo-sdk`, which lives in **AWS CodeArtifact**, not PyPI. Installing needs an auth token valid 12
hours, obtained by `./scripts/setup.sh`.

**Why this matters operationally:** every developer and every build machine needs AWS credentials with
CodeArtifact read access just to install dependencies. The Terraform in `ec2_iam.tf` grants the EC2
instance profile that access (commit `9db841d`). It's also a genuine CI friction point — a build
pipeline needs a role assumption before `uv sync`, and a 12-hour token means you can't cache
credentials across a day.

It even affected the token optimization work: `_token_report.md` records that tiktoken couldn't be
installed because of CodeArtifact pip auth, which is why token counts are a `chars/4` heuristic.
**That's a nice, concrete example of infrastructure constraints shaping engineering decisions** —
worth mentioning.

### Migrations

Alembic, with `psycopg2-binary` present purely because Alembic runs synchronously while the app is
async. Standard flow: `alembic revision --autogenerate -m "..."` → review → `alembic upgrade head`.

Migration history shows real production discipline — dropping the `tokens` table when per-user
providers replaced shared OAuth, dropping `project_artifacts` when files moved to path convention,
dropping `cap_broker_*` columns when the brokerage capability was removed, and backfilling
`email_verified` for existing users when signup changed.

**The interview-relevant point:** `--autogenerate` diffs models against the database, but it doesn't
reliably detect column renames — it emits drop+add, which destroys data. Review every generated
migration. Also, migrations against a live table need care: adding a `NOT NULL` column without a
default locks and fails on a populated table; the safe pattern is nullable → backfill → constrain.

### Testing, precisely

| Layer | Status |
|---|---|
| API endpoints | **Manual** — Postman collections. `CLAUDE.md`: "Manual Testing Only — No automated tests." |
| Agent behaviour | **Automated** — `quantjo-agent-tests`, 20 multi-turn cases, 3 assertion layers |
| Harness infrastructure | Partly — a `no_llm_required` pytest marker exists for tests that don't need a real LLM |
| CI | **None found** — no workflow files |

The harness ships two entry points: a Typer CLI (`quantjo-agent-test`) and a pytest plugin registered
via `[project.entry-points."pytest11"]`, with a `--repetitions` flag. So it's *CI-ready* — it just
wasn't wired to a gate.

**Say it that way.** "It's runnable in CI via the pytest plugin; wiring it into a deploy gate was the
next step" is accurate and still credits the work.

### The deploy-time gate that was designed but not wired

The harness's stated purpose is "deploy-time multi-turn eval." The intended shape:

```
PR touching agent/  →  deploy to staging  →  quantjo-agent-test run --repetitions=3
                                              ├─ blocking diffs → fail
                                              ├─ Layer 1 failure → fail
                                              └─ green → --update-baseline → promote
```

Everything needed exists — baseline storage, blocking vs advisory classification,
`get_blocking_diffs`, JSON reports. What's missing is the workflow file and a staging environment
dedicated to it. Being able to describe the gate you designed, and honestly note it wasn't wired, is
better than implying it ran on every PR.

### Deployment reality

EC2-based, `ap-south-1`, documented in `docs/EC2_SETUP.md` and `docs/INSTALLATION.md`. HTTPS via
Let's Encrypt is **mandatory** because Zerodha's OAuth callback requires it — a third-party constraint
driving infrastructure.

Terraform covers state bootstrap, the dev environment, IAM, and a `trading_ecs` module. The
paper-trading container is built and pushed by `scripts/push_trading_image.sh` and runs on ECS —
so containerized workloads exist as a pattern, even though the API itself runs on EC2.

**The gap:** no documented blue/green or rolling deploy. With persistent WebSocket connections that's
a real problem — restarting the process drops every in-flight agent turn. Proper handling means
draining: stop accepting new connections, let in-flight turns finish, then terminate. That's a good
unprompted observation.

### Monitoring

**What exists:**
- `DebugLogger` — `log_request_start`, `log_request_end` with duration, `log_phase` context manager
  for timing, `log_guardrail_check`, `log_agent_response`, `log_error` with context. That's
  structured and genuinely useful.
- `token_usage` — per-message rows with prompt/completion/total/cached tokens, model, provider,
  operation, intent. Excellent raw material for cost attribution.
- `scripts/usage_report.py` for beta reporting.

**What's missing:** APM, distributed tracing, metrics/dashboards, alerting, error aggregation
(Sentry), and log aggregation. Some lifecycle logging still uses `print()` in
`connection_manager.py`.

**Where I'd start:** OpenTelemetry traces spanning WebSocket → agent turn → tool calls → OpenAI → S3.
A 30–60 second turn is currently a black box in aggregate; a trace makes latency attributable to a
specific tool or model call. Then alerting on turn error rate, judge-error rate, and quota rejections.

---

## 5. My Actual Contribution

**What existed** — local dev setup, `uv` + CodeArtifact tooling, Alembic migrations, Postman
collections, Terraform, EC2 deployment, `DebugLogger`, `token_usage`.

**What I built** — the automated testing capability: `quantjo-agent-tests` as an installable package
with a CLI and pytest plugin, report generation (JSON + markdown), and baseline storage with
regression classification. Plus `scripts/compile_sdk_wiki.py` as a build-time step for the knowledge
base, with `--dry-run` / `--plan-only` / `--from-plan` staging.

**What I did not do** — CI pipelines, deployment automation, infrastructure provisioning, or
monitoring setup. **State this plainly.**

**Result** — the agent layer became testable and its quality trackable across runs; the KB gained a
reproducible, reviewable build step.

---

## 6. Files and Code

| Area | Path |
|---|---|
| Dev setup | `scripts/setup.sh`, `scripts/seed_dev_data.py`, `scripts/seed_templates.py`, `scripts/create_user.py` |
| Dependencies | `pyproject.toml`, `uv.lock` |
| Migrations | `alembic.ini`, `alembic/versions/` |
| Manual API testing | `postman/collections/`, `postman/environments/` |
| Agent testing | `quantjo-agent-tests/` — `cli.py`, `pytest_plugin.py`, `reports/`, `baselines/` |
| KB build step | `scripts/compile_sdk_wiki.py` |
| Infrastructure | `infra/terraform/bootstrap/`, `dev/main.tf`, `dev/ec2_iam.tf`, `modules/trading_ecs/` |
| Container | `scripts/push_trading_image.sh`, `.dockerignore` |
| Ops scripts | `usage_report.py`, `purge_project_data.py`, `purge_user_data.py`, `backfill_head_files.py`, `stop_free_paper_trades.py`, `clean_dead_ai_state_keys.py` |
| Docs | `docs/EC2_SETUP.md`, `docs/INSTALLATION.md`, `README.md`, `CLAUDE.md`, `ARCHITECTURE.md` |
| Process | `.github/CODEOWNERS`, `PULL_REQUEST_TEMPLATE.md`, `PULL_REQUEST_PROCESS.md`, `CONTRIBUTING.md` |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| `uv` over pip/poetry | Fast resolution, lockfile | Less ubiquitous; team learning curve |
| CodeArtifact for the SDK | Private package, IAM-controlled | 12h tokens; CI friction; blocked tiktoken |
| Manual API testing | Ship features first | No regression safety on the REST surface |
| Automated agent testing | The AI layer is the least verifiable part | Slow, costs tokens, needs a live server |
| Separate test repo | Tests the deployed system, not the source | Version compatibility to manage |
| Terraform | Reproducible infra, reviewable in PRs | Learning curve; state management |
| EC2 for the API | Simple, direct control | Manual scaling; manual deploys |
| ECS for the trading container | Isolation for long-running workloads | Two deployment models to maintain |
| Offline KB compile | Reviewable in a PR; no request-time cost | Manual step that can be skipped |
| `DebugLogger` over APM | No vendor dependency, zero cost | No aggregation, dashboards, or alerts |

---

## 8. Production Improvements

1. **CI pipeline** — lint (ruff), type-check (mypy), format (black), and unit tests on every PR. All
   four tools are already dev dependencies and unused in automation.
2. **Deploy gate on the agent harness** — run against staging on any PR touching `agent/` or
   `agent_kb/`; block on `get_blocking_diffs`.
3. **API integration tests** — the biggest coverage gap. Postman collections could seed a
   `pytest` + `httpx` suite against a test database.
4. **KB staleness check in CI** — hash `raw/`, fail if `.compile-state.json` disagrees.
5. **Containerize the API** and deploy on ECS/EKS for consistent artifacts and rollback.
6. **Blue/green or rolling deploys with connection draining** — mandatory given persistent
   WebSockets.
7. **OpenTelemetry** end-to-end tracing.
8. **Error aggregation** (Sentry) and log shipping (CloudWatch/Datadog).
9. **Alerting** on turn error rate, judge error rate, quota rejections, WS disconnect rate, and
   navigator-fallback events (which signal a silent 25x cost regression).
10. **Migration safety checks** — flag destructive operations in review; enforce
    nullable → backfill → constrain.
11. **Staging environment** mirroring production, which the deploy gate needs anyway.

---

## 9. Common Interview Questions

1. Walk me through local development setup.
2. Why `uv`?
3. How is the private SDK distributed and what does that cost you?
4. How do migrations work?
5. What are the risks of `--autogenerate`?
6. How was the API tested?
7. What did you automate?
8. Do you have CI? *(Answer honestly: no workflow files.)*
9. How would you wire the harness into a deploy gate?
10. How is the application deployed?
11. Why is HTTPS mandatory?
12. How do you deploy without dropping WebSocket connections?
13. What's in Terraform and what isn't?
14. Why EC2 for the API but ECS for the trading container?
15. What monitoring exists?
16. How would you debug a slow agent turn in production?
17. What would you alert on?
18. How do you track AI cost per user?
19. What's your rollback story?
20. What's the biggest operational gap?

---

## 10. Strong Answers

**Q8 — Do you have CI?**
> Not as workflow files — I want to be accurate about that. The repos have `CODEOWNERS`, a PR
> template, and a documented PR process, so there's review discipline, but no automated pipeline. The
> tooling is all there and unused in automation: ruff, mypy, black, and pytest are dev dependencies.
> What I built was CI-*ready* rather than CI-wired — the agent harness ships a Typer CLI and a pytest
> plugin with a `--repetitions` flag, and it writes JSON reports plus a baseline with blocking versus
> advisory classification. So the gate logic exists, `get_blocking_diffs` is the function that would
> fail a build, and what's missing is the workflow file and a dedicated staging environment. That was
> the next step and I'd rather say so than imply it ran on every PR.

**Q12 — Deploying without dropping connections.**
> This is a real problem and it isn't solved today. Restarting the process kills every WebSocket, and
> because agent turns run as in-process background tasks, any turn in flight is lost — the user sees
> the connection drop mid-generation and has already been charged tokens for work that produced
> nothing. Proper handling is connection draining: on SIGTERM, stop accepting new connections, stop
> accepting new messages on existing ones, let in-flight turns finish with a bounded grace period,
> then terminate. That pairs with a rolling deploy so the load balancer moves new connections to the
> new instances while old ones drain. The deeper fix is making turns resumable — persisting turn state
> so a reconnecting client can pick up a completed response rather than losing it. The idempotency
> mechanism we already have on `message_id` is most of the way there, since a retry returns the cached
> response.

**Q16 — Debugging a slow turn in production.**
> Today it's painful, and I'd say that directly. `DebugLogger` gives per-phase timing with a
> `log_phase` context manager and a `message_id` to correlate on, and `token_usage` tells me the token
> counts and which model ran. So I can tell you a turn took 90 seconds and used 40,000 tokens. What I
> can't tell you from logs is *where* those 90 seconds went — how much was OpenAI, how much was the
> Codex sub-run, how many SDK pages it fetched, how much was S3. That needs distributed tracing. A
> single trace per turn with spans for each tool call, each model call, and each S3 operation would
> turn a black box into an attributable latency breakdown, and it's the first thing I'd add. With
> traces you can also answer the aggregate question — is the p99 driven by one slow tool or by a long
> tail of retries — which logs simply can't.

**Q20 — Biggest operational gap.**
> Observability, and it's not close. The system's core interaction takes thirty to sixty seconds,
> costs real money, depends on a third-party API, and is currently only visible through log lines. No
> tracing, no metrics, no alerting, no error aggregation. Concretely: if the SDK navigator failed to
> build, the code silently falls back to injecting the full twenty-thousand-token guide — that's a
> roughly twenty-five-times cost regression on the code path, and today it's a `logger.warning` that
> nobody would see. That's exactly the kind of failure that should page someone, and it's a good
> illustration of why "it degrades gracefully" isn't sufficient without monitoring to tell you it
> degraded.

---

## 11. Follow-up Questions

- "No CI — how did you know a change didn't break the build?"
- "12-hour CodeArtifact tokens. How would that work in a CI pipeline?"
- "Terraform covers dev. What about production state and drift?"
- "You said the harness needs a live server. Which environment would the gate run against?"
- "How do you roll back a bad migration that already ran on the beta?"
- "What's your data backup and restore story?"
- "How would you detect the navigator fallback firing in production?"

---

## 12. Senior Engineer Perspective

**They'll assess whether you know the difference between "we didn't have it" and "we didn't need it."**
For a small-team beta, no CI is a defensible tradeoff — but only if you can say what it cost and when
it stops being acceptable.

**They'll value that you built the hardest test.** The API layer is the *easy* thing to test and it
wasn't tested; the agent layer is the hard thing and you built a real harness for it. That's inverted
from what most people do, and it's the right prioritization because the agent was the risky, novel,
unverifiable part.

**They'll probe operational maturity.** Connection draining, migration safety, rollback, and
observability are where seniority shows. Having thought-through answers on things you *didn't* build
is fine — pretending you built them is not.

**The navigator-fallback alerting point is a strong closer.** It connects your own feature to an
operational gap and shows you think about your code after it ships.

---

## 13. Common Mistakes

- **Inventing a CI/CD pipeline.** There are no workflow files. This is trivially checkable and it's
  the kind of lie that ends an interview.
- **Saying "we had tests" without qualification.** The agent layer had automated tests; the API layer
  was manual Postman. Be precise — the precision is credible.
- **Claiming monitoring you don't have.** Structured logging and a token-usage table is what exists.
- **Not knowing what you'd add first.** Observability. Have the specific answer.
- **Overlooking WebSocket draining.** It's the deployment problem unique to this architecture and
  most candidates miss it entirely.

---

## 14. Cheat Sheet

- **Dev:** `uv` + lockfile · SDK from **AWS CodeArtifact** (12h token, `./scripts/setup.sh`) ·
  `createdb quantjo_dev` → `alembic upgrade head` → `seed_dev_data.py` ·
  `uv run uvicorn quantjo_api.main:app --port 8000 --reload` · `npm run dev` :3000.
- **Testing:** API = **manual Postman**. Agent = **automated**, `quantjo-agent-tests` (Typer CLI +
  `pytest11` plugin, `--repetitions`), reports JSON+MD, baselines with blocking/advisory diffs.
  **No CI workflow files exist.**
- **Build step:** `scripts/compile_sdk_wiki.py --dry-run | --plan-only | --from-plan`.
- **Infra:** Terraform — `bootstrap/` (state), `dev/` (`main.tf`, `ec2_iam.tf`), `modules/trading_ecs/`.
  API on **EC2** (ap-south-1, instance profile); paper-trading container on **ECS** via
  `push_trading_image.sh`. **HTTPS mandatory** (Zerodha OAuth callback).
- **Monitoring:** `DebugLogger` (request start/end, `log_phase` timing, guardrail checks, errors) +
  `token_usage` table (prompt/completion/total/**cached**, per user/project/model) +
  `usage_report.py`. **No APM, tracing, alerting, or error aggregation.** Some `print()` remains.
- **Add first:** OpenTelemetry traces per turn → alerting (turn errors, judge errors, quota
  rejections, **navigator fallback** = 25× cost regression) → CI gate on the harness → API
  integration tests → connection-draining deploys.
- **Honest line:** "I built the agent test harness and the KB build step. I didn't build CI,
  deployment, or monitoring."
