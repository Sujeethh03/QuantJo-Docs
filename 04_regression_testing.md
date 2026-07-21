# Agentic Regression Test Harness

**Your second headline contribution. This is the most "senior" thing in the internship — lead with
it when an interviewer asks about engineering rigor.**

---

## 1. Interview Answer (2–3 minutes)

> The problem I was solving is that we had no way to know if a prompt change made the agent worse.
> Our agent writes trading strategy code, and its behaviour is governed by a system prompt. Someone
> tweaks one sentence to fix a bug in exit handling, ships it, and silently breaks pairs trading for
> every user. There's no compiler error. There's no failing unit test. You find out from a support
> ticket, or you don't find out at all.
>
> So I built `quantjo-agent-tests` — a standalone package that runs twenty multi-turn conversations
> against a live server and scores the results. It's not a mock. It creates a real project over REST,
> opens a real WebSocket, and talks to the actual agent, because the thing I need to catch is
> integration-level behaviour, not unit-level.
>
> The conversations are driven by an LLM user simulator. Each case defines a persona and a goal —
> "experienced trader, terse, wants a 20/50 SMA crossover with a 2% stop." The simulator plays the
> human, and critically it only sees the agent's text replies, not its tool calls, which mirrors what
> a real user experiences. It emits a `[DONE]` sentinel when its goal is met so conversations
> terminate naturally rather than always running to max turns.
>
> Scoring is three layers. Layer 1 is deterministic AST assertions using Python's `ast` module —
> does it subclass `Strategy`, does `generate_signals` return, are there any `eval` or `subprocess`
> calls. Those are structural facts, so they're cheap and they never flake. Layer 2 is an LLM judge
> scoring four dimensions — intent capture, SDK correctness, exit completeness, conversation quality —
> at temperature zero with a forced JSON schema. Layer 3 is the part I'm proudest of: boundary probe
> pairs. For each of six confusable SDK concepts, I have two cases. Probe A must exhibit pattern A
> and must *not* exhibit pattern B, and vice versa. That catches over-correction — the failure where
> you fix one behaviour and the agent starts applying it everywhere.
>
> On top of that: each case runs k times with a majority vote, so a split verdict is reported as
> "flaky" rather than randomly passing or failing. And every run diffs against a stored baseline, so
> a required judge dimension dropping more than 0.15 is a blocking regression even if the case still
> technically passes.

---

## 2. What Problem Were We Solving?

### Business problem

The agent's output is code a trader may risk money on. Silent quality degradation is the worst
possible failure: nobody notices, and the damage is financial and reputational. Meanwhile prompt
changes were frequent — the agent layer was under active development — and each one had unbounded
blast radius across every user.

### Technical problem

**LLM systems are not testable by conventional means.** Every assumption unit testing relies on
breaks:

| Conventional testing assumes | LLM reality |
|---|---|
| Deterministic output | Same input → different output every run |
| Exact assertions | `assert output == expected` is meaningless |
| Local failures | One prompt word changes behaviour globally |
| Binary pass/fail | Quality is continuous and multi-dimensional |
| Fast, cheap tests | Each case is a 60s multi-turn conversation costing real tokens |
| Single-turn | Real usage is a negotiation over several turns |

### Why existing approaches weren't enough

- **Unit tests on prompt strings** verify the string, not the behaviour. Worthless here.
- **Golden-output snapshots** fail on every run because generation is stochastic.
- **Manual QA** doesn't scale, isn't reproducible, and nobody does it consistently before a
  one-line prompt fix.
- **Single-turn eval sets** miss the entire class of conversational failures — asking redundant
  questions, generating too early, losing context established three turns back.
- **Generic eval frameworks** (DeepEval, Ragas) score text quality but know nothing about whether
  `ExitRule` was used correctly for a fixed-percentage stop.

---

## 3. Architecture

```
                    quantjo-agent-test run --judge=openai --repetitions=3
                                    │
                    ┌───────────────▼────────────────┐
                    │  cli.py (typer)                │
                    │  config.py (pydantic-settings) │
                    └───────────────┬────────────────┘
                                    ▼
                    ┌────────────────────────────────┐
                    │  case_loader.py                │
                    │  20 YAML cases → CaseSpec       │
                    │  plugins.py loads quantjo_plugin│
                    │   (SDK predicates + boundaries) │
                    └───────────────┬────────────────┘
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │  runner.py :: execute_case()                      │
        │    for rep in range(repetitions):                 │
        │        asyncio.run(_execute_v2_async(...))        │
        │    → _aggregate_reps()   [majority vote]          │
        └───────────────────────┬───────────────────────────┘
                                ▼
   ┌──────────────────────────────────────────────────────────┐
   │  MULTI-TURN CONVERSATION LOOP                            │
   │                                                          │
   │  AgentWebSocketClient          UserSimulator (gpt-4o)    │
   │    POST /projects                  persona + goal        │
   │    WS /ws/projects/{id}                 │                │
   │    {"type":"auth","token":jwt}          │                │
   │         │                               │                │
   │    ┌────▼───────────────────────────────▼─────┐          │
   │    │ turn 1: opening_message ────► agent      │          │
   │    │         agent reply ────────► simulator  │          │
   │    │ turn 2: simulated reply ────► agent      │          │
   │    │         ...                              │          │
   │    │ stop on: agent_proposes_files            │          │
   │    │        | [DONE] sentinel                 │          │
   │    │        | max_turns                       │          │
   │    │        | agent_error                     │          │
   │    └────┬─────────────────────────────────────┘          │
   │         │ transcript + proposed_files["backtest.py"]     │
   └─────────┼──────────────────────────────────────────────────┘
             ▼
   ┌─────────────────────────────────────────────────────────┐
   │ LAYER 1 — assertions/layer1.py    deterministic, free   │
   │   ast.parse() → predicate registry                      │
   │   defines_strategy_subclass, has_generate_signals,      │
   │   safe_python, + 19 SDK-specific from the plugin        │
   │   ANY failure → hard fail (short-circuits the case)     │
   └────────────────────────┬────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │ LAYER 2 — assertions/layer2.py    semantic, costs money │
   │   modes: openai | deepeval | shadow | off               │
   │   gpt-4o-mini, temperature=0.0, response_format=json    │
   │   dims: intent_capture, sdk_correctness,                │
   │         exit_completeness, conversation_quality         │
   │   required_pass → gates.  advisory → warns only.        │
   │   waivers → suppress a known-bad dim with an expiry     │
   └────────────────────────┬────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │ LAYER 3 — assertions/layer3.py    cross-case            │
   │   for each boundary B1..B6:                             │
   │     A must pass pred_A and FAIL pred_B                   │
   │     B must pass pred_B and FAIL pred_A                   │
   │   crossover_clean = all four conditions hold            │
   └────────────────────────┬────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────┐
   │ baseline.py — diff vs baselines/last_green.json         │
   │   result flip            → blocking                     │
   │   required dim drop≥0.15 → blocking                     │
   │   pass→unsure/fail       → advisory_regression          │
   │ reporter.py + scorecard.py → reports/{ts}.json + .md    │
   └─────────────────────────────────────────────────────────┘
```

---

## 4. Deep Technical Explanation

### The 20 cases — deliberate composition

| Class | Count | Directory | Purpose |
|---|---|---|---|
| **positive** | 5 | `cases/positive/` | Canonical strategies that must work: SMA golden cross, EMA+RSI, RSI mean reversion, BB squeeze breakout, pairs trading |
| **probe** | 12 | `cases/probes/` | Six boundary pairs (B1–B6), A and B each |
| **qc** | 3 | `cases/qc/` | Known correctness traps: VWAP session reset, stop direction inversion, entry/exit zone overlap |

Not twenty arbitrary tests. Five prove the happy path, twelve probe the ambiguous seams documented in
the KB's `INDEX.md`, three encode specific bugs that had bitten before. **When asked "why twenty?",
that breakdown is the answer.**

### The user simulator — why it exists and why it's constrained

```python
class UserSimulator:
    def __init__(self, persona: str, goal: str, *, model="gpt-4o",
                 temperature=0.0, max_tokens=256)
```

*Why simulate at all?* Because single-turn eval misses the failures that matter. Real traders don't
paste a complete spec — they say "I want an RSI thing," the agent asks a clarifying question, they
answer partially. Failures live in that negotiation: asking five questions instead of one, generating
before confirming, forgetting the timeframe established two turns ago.

*Why does the simulator not see tool calls?* `_strip_tool_internals()` filters out `function_call`,
`function_call_output`, `tool_call`, `tool_result`. This mirrors tau-bench's design and, more
importantly, **reflects reality** — a real user sees text, not the agent's internals. If the
simulator could see tool calls it would react to information the real user never has, and you'd be
testing a fictional interaction.

*Why role flipping?* In `_build_messages`, the transcript's roles are inverted: the agent's `user`
turns become `assistant` from the simulator's perspective. The simulator is a distinct LLM playing
the other side of the conversation.

*Why the `[DONE]` sentinel?* Without it every case runs to `max_turns`, burning tokens and time and
polluting the transcript with filler the judge then scores. The sentinel lets a conversation end when
its goal is met. Note the defensive detail: an empty reply returns `DONE_SENTINEL` rather than hanging.

*Why temperature 0.0?* The system under test should be the only source of variance.

*Why the aggressive prompt constraints?* "NEVER ask a question." "NEVER repeat, paraphrase, or
confirm your own previous statement." "Do NOT reveal that you are a simulator." Each is a patched
failure mode — simulators drift into assistant behaviour, echo themselves, and break character.

### Layer 1 — deterministic AST assertions

Covered in depth in [05](05_ast_validation.md). Key architectural points:

- **Registry pattern.** `register_predicate(name, fn)` with a central `_REGISTRY` dict. Generic
  predicates live in the core; SDK-specific ones ship in `quantjo_plugin` via a pytest entry point.
  That boundary keeps the harness reusable for any agent, not just this one.
- **Uniform signature.** `fn(tree, *, source, **kwargs) -> tuple[bool, str]` — every predicate
  returns a verdict *and evidence*. A failing test that doesn't tell you why is nearly useless.
- **Fail fast on unknown predicates** — a typo'd predicate name is a config error, not a silent skip.
- **Predicate exceptions are caught** and converted to `(False, "predicate error: ...")` so one buggy
  predicate can't crash a 20-minute run.

### Layer 2 — the LLM judge

Covered in depth in [06](06_llm_judge.md). Architectural points:

- **Four modes.** `openai` (default), `deepeval`, `shadow`, `off`. **Shadow mode is the standout
  design decision**: it runs both backends, the OpenAI verdict gates pass/fail, and the DeepEval
  verdict rides along as `shadow_verdict` for comparison. That's how you migrate an evaluator without
  trusting it first — the same pattern as shadow-traffic testing a new service.
- **`required_pass` vs `advisory`.** Per case. `conversation_quality` is almost always advisory
  because it's the most subjective dimension; `intent_capture` gates.
- **Waivers with expiry.** `{dimension, reason, expires}` — a known-bad dimension can be suppressed
  with a reason and a date, instead of deleting the test. This is how you keep a suite green without
  losing signal.
- **Judge errors fail *open*.** `_error_result` returns `layer2_passed: True` with a warning. A
  flaky judge API must not block a deploy on a false negative. Layer 1 still gates hard.

### Layer 3 — boundary probes and over-correction

The most original piece. Consider B1: declarative vs. manual indicators. Probe A asks for a standard
SMA (must use `.config()`); Probe B asks for a Hull MA (must be computed manually — no SDK builtin).

```python
a_own_passed  = check(a_code, pred_a)   # A does A          ✓
b_own_passed  = check(b_code, pred_b)   # B does B          ✓
a_failed_other = not check(a_code, pred_b)  # A does NOT do B  ✓
b_failed_other = not check(b_code, pred_a)  # B does NOT do A  ✓
crossover_clean = all four
```

*Why does this matter?* Because the characteristic failure of prompt engineering is **over-correction**.
You notice the agent doesn't compute Hull MA manually, so you add "compute indicators manually when
the SDK lacks them." Now it computes SMA manually too — you fixed B and broke A. A conventional test
suite where each case only asserts its own behaviour would show B going green and A staying green
(A's own assertions still pass), and you'd ship the regression. The crossover check is what catches
it.

B4 carries a special case in code: probe A is the *negative* case (no session filter should appear),
so its logic is inverted explicitly rather than shoehorned into the generic pattern. Worth mentioning
— it shows you handled a case the abstraction didn't cover.

### Majority voting — handling nondeterminism honestly

```python
half = (n + 1) // 2
if   n_pass >= half: verdict = "pass"
elif n_fail >= half: verdict = "fail"
else:                verdict = "flaky"
```

*Why a third state?* Because collapsing a 2-pass/2-fail run into either "pass" or "fail" throws away
the most important information: **the behaviour is unstable**. An agent that produces correct code 60%
of the time is a different (and worse) product than one that's correct 100% or 0% of the time. The
`flaky` verdict surfaces that, and `rep_verdicts` plus per-rep summaries in `reps[]` let you inspect
the split.

Note it's `pass^k`-flavoured, not `pass@k`: it asks "is this reliably right," not "is it ever right."
For a code generator a user depends on, reliability is the correct question.

### Baseline diffing — the actual regression detection

Three diff classes in `compute_diff()`:

| Condition | Type | Effect |
|---|---|---|
| `result` flipped to `fail` | **blocking** | Fails the run |
| Required dim score dropped ≥ `0.15` | **blocking** | Fails the run |
| Dim went `pass` → `unsure`/`fail` | advisory | Marks `advisory_regression` |
| Case is new | info | Reported only |
| Waiver removed | info | Shows the now-unmasked score |

**The score-drop check is the subtle, valuable one.** A case can still "pass" while
`sdk_correctness` slides from 0.93 to 0.71. Binary pass/fail hides that; a 0.15 threshold on required
dimensions catches quality erosion before it crosses into failure. That's the difference between a
regression suite and a smoke test.

One deliberate change documented in `baseline.py`: `update_baseline` saves **unconditionally** — the
baseline is "last known state," not "last green state." `is_green()` survives only as a reporting
label. *Why?* If you only save green baselines, a legitimately-changed expectation leaves you
diffing against an ancient run forever, and the diff becomes noise everyone ignores.

### Why a separate repository

`quantjo-agent-tests` has its own `pyproject.toml`, its own venv, its own CLI (`quantjo-agent-test`),
and registers a pytest plugin via `[project.entry-points."pytest11"]`. It depends on the API only
over the network — no imports from `quantjo_api`.

*Why?* Because it tests the **deployed system**, not the code. It can run against local, staging, or
production. If it imported the app it would test the code in the repo, which is precisely the thing
that might be wrong.

---

## 5. My Actual Contribution

**What existed** — nothing. `quantjo-saas/CLAUDE.md` said plainly: "Manual Testing Only — No
automated tests." Two git commits (`d5b3e9c` "agentic-testing", `f9894e6`
"user-message-based-test-simulation") mark the work.

**What I built** — the whole package:
- `client/` — `AgentWebSocketClient` speaking the real WS protocol; `protocol.py` with the
  `AgentClient` interface, `TurnResult`, `ToolCallEvent`
- `harness/user_simulator.py` — persona-driven simulator with tool-internal stripping and `[DONE]`
- `harness/runner.py` — orchestration, multi-turn loop, changeset policies, rep aggregation
- `harness/ast_utils.py` — AST helper layer
- `assertions/layer1.py`, `layer2.py`, `layer3.py` — the three scoring layers
- `harness/judge/` — `prompt.py`, `client.py`, `schema.py`, `deepeval_client.py`
- `harness/baseline.py` — diffing and regression classification
- `harness/reporter.py`, `scorecard.py` — JSON + markdown reports
- `harness/case_loader.py`, `config.py`, `plugins.py`, `cli.py`, `pytest_plugin.py`
- `plugins/quantjo_plugin/` — 19 SDK-specific predicates + B1–B6 boundary registrations
- 20 case YAMLs

**What changed** — prompt and agent changes became verifiable. Reports in `reports/` show real runs
with per-dimension scores and full conversation transcripts.

**Result** — a deploy-time quality gate: three scoring layers, majority voting over repetitions,
baseline regression detection with a 0.15 score-drop threshold, and over-correction detection that
conventional per-case testing structurally cannot provide.

---

## 6. Files and Code

| Path | Contents |
|---|---|
| `cli.py` | `quantjo-agent-test` typer CLI |
| `config.py` | `Settings` — `ws_url`, `api_url`, `jwt`, `openai_api_key` |
| `pytest_plugin.py` | pytest integration; `--repetitions` flag |
| `plugins.py` | Entry-point plugin discovery |
| `client/protocol.py` | `AgentClient`, `TurnResult`, `ToolCallEvent` |
| `client/websocket_client.py` | `AgentWebSocketClient` — `connect`, `send_user_message`, `accept_changeset`, `reject_changeset`, `_await_turn_complete`; `_MAX_FRAMES=500`, `_IGNORED_TYPES` |
| `harness/case_loader.py` | YAML → `CaseSpec`, `Scenario`, `IntentSpec`, `JudgeSpec`, `Waiver` |
| `harness/user_simulator.py` | `UserSimulator`, `_strip_tool_internals`, `DONE_SENTINEL` |
| `harness/runner.py` | `execute_case`, `_execute_v2_async`, `_aggregate_reps`, `_make_result` |
| `harness/ast_utils.py` | `find_class`, `find_method`, `class_attr_value`, `walk_calls`, `call_name`, `assigned_column_names`, `has_for_loop_with_range` |
| `assertions/layer1.py` | `_REGISTRY`, `register_predicate`, `run_predicate`, `run_all_predicates` |
| `assertions/layer2.py` | `run_layer2`, `_build_result`, `_error_result` |
| `assertions/layer3.py` | `_BOUNDARY_MAP`, `register_boundary_pair`, `run_pair_check`, `PairResult`, `collect_pair_results` |
| `harness/judge/prompt.py` | `SYSTEM_PROMPT`, `CONVERSATION_TEMPLATE`, `build_conversation_message` |
| `harness/judge/client.py` | `run_judge_conversation`, `_call_openai`, `_validate_verdict`, `_normalize_scores` |
| `harness/judge/schema.py` | `V1_DIMENSIONS`, `V2_DIMENSIONS`, thresholds, `derive_verdict` |
| `harness/baseline.py` | `load_baseline`, `compute_diff`, `mark_advisory_regressions`, `update_baseline`, `get_blocking_diffs` |
| `plugins/.../predicates.py` | 19 SDK predicates |
| `plugins/.../boundary_pairs.py` | B1–B6 registrations |
| `baselines/last_green.json`, `last_state.meta.json` | Stored baseline |
| `reports/{timestamp}.json` / `.md` | Run output |

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| Separate repo, network-only coupling | Tests the deployed system, not the source | Needs a running server; can't run in a pure unit-test CI |
| Live WebSocket, not mocks | Catches integration failures | Slow (~60s/case), costs tokens, needs credentials |
| LLM user simulator | Multi-turn failures are the interesting ones | Adds a second source of nondeterminism |
| Simulator blind to tool calls | Matches real user experience | Can't assert on tool-level behaviour from the user side |
| Three layers, cheapest first | Layer 1 short-circuits before paying for the judge | More machinery to maintain |
| Layer 1 gates hard, Layer 2 gates selectively | Structure is objective; semantics are not | Semantic regressions can slip past as advisory |
| Judge errors fail open | Flaky judge API must not block deploys | A persistently broken judge silently stops gating |
| Majority vote with `flaky` state | Preserves the instability signal | Cost multiplies by k |
| Baseline = last state, not last green | Prevents diffing against an ancient run forever | A bad run becomes the new baseline if not reviewed |
| 0.15 score-drop threshold | Catches erosion before failure | Threshold is a judgment call, not derived |
| Waivers with expiry | Suppress known issues without deleting tests | Expired waivers need enforcement |
| Plugin architecture | Core harness is agent-agnostic | Indirection |

---

## 8. Production Improvements

1. **Statistical rigor on the threshold.** 0.15 is intuition. With enough runs you can measure the
   judge's variance on identical input and set the threshold at ~2σ.
2. **Judge-vs-human calibration.** Nobody has verified the judge agrees with expert review. Label
   100 outputs by hand, measure agreement (Cohen's κ), and report it. Without this the judge is an
   unvalidated instrument.
3. **Parallel case execution.** Cases are independent; the run is currently serial. 20 cases × 60s ×
   k reps is slow enough to discourage running it.
4. **CI integration on a schedule.** Nightly against staging, plus on PRs touching `agent/`.
5. **Cost tracking per run.** `aggregate_tokens` is initialized but stays zero — the WS client
   doesn't consume the `token_usage` event the server emits. That's a real, small, fixable gap.
6. **Deterministic seeding** where the provider supports it, to reduce k.
7. **Case coverage as a first-class metric.** 20 cases against 15 topics — which topics are untested?
8. **Auto-expire waivers** — fail the build when `expires` has passed.
9. **Trend dashboards** — per-dimension score over time reveals slow erosion that pairwise diffing
   misses.
10. **Adversarial cases** — prompt injection via strategy descriptions, contradictory requirements,
    requests for unsupported asset classes.

---

## 9. Common Interview Questions

1. Why can't you unit-test an LLM agent?
2. Walk me through what one test case does end to end.
3. Why twenty cases? Why those twenty?
4. What is a boundary probe and what does it catch?
5. What's over-correction and why does normal testing miss it?
6. How do you handle nondeterminism?
7. Why three verdicts instead of pass/fail?
8. Why does Layer 1 exist if you have an LLM judge?
9. Why do judge errors fail open?
10. How do you detect a regression if the test still passes?
11. Why is the baseline "last state" rather than "last green"?
12. What's a waiver and why not just delete the test?
13. Why does the user simulator not see tool calls?
14. What's the `[DONE]` sentinel for?
15. Why a separate repository?
16. What's shadow judge mode?
17. How long does a full run take and what does it cost?
18. What's the flakiest part of the suite?
19. How do you know the judge is right?
20. If a case fails, how do you tell whether the agent or the test broke?

---

## 10. Strong Answers

**Q1 — Why can't you unit-test it?**
> Because every assumption unit testing relies on is false. Output isn't deterministic, so exact
> assertions don't work. Failures aren't local — one word in a system prompt changes behaviour across
> every case, so there's no unit to isolate. And quality is continuous, not binary; code can be
> syntactically perfect, structurally valid, and still implement the wrong strategy. What you can do
> is assert the properties that *are* deterministic — structure, via AST — and score the rest with a
> judge, then track both against a baseline so you're measuring change rather than absolute
> correctness. That reframing, from "is it correct" to "did it get worse," is what made the problem
> tractable.

**Q5 — Over-correction.**
> It's the characteristic failure of prompt engineering. Say the agent isn't computing a Hull moving
> average manually when it should — the SDK has no built-in. You add a line: "compute indicators
> manually when the SDK lacks them." Now it computes SMA manually too, even though there's a built-in
> `.config()` API for it. You fixed one case and broke another. Normal per-case testing misses this
> because the SMA case's own assertions still pass — it *does* produce a working SMA strategy, just
> the wrong way. My Layer 3 pairs each boundary: probe A must exhibit pattern A and must *not* exhibit
> pattern B, and probe B the inverse. Four conditions, all required for `crossover_clean`. That
> negative assertion — "A must not do B" — is what catches the regression.

**Q10 — Regression when the test still passes.**
> That's the baseline score-drop check, and it's the part I think is most valuable. Each case has
> required judge dimensions. If `sdk_correctness` was 0.93 on the baseline and comes back 0.71 today,
> the case might still pass — 0.71 is above the 0.7 pass threshold — but I flag it as blocking because
> the drop exceeds 0.15. Binary pass/fail would show green while quality erodes underneath you. I'd be
> honest that 0.15 is a judgment call rather than a statistically derived number; the rigorous version
> is to measure the judge's variance on identical input and set the threshold at about two standard
> deviations.

**Q9 — Judge errors fail open. Isn't that dangerous?**
> It's a deliberate tradeoff and I'd defend it, with a caveat. If the OpenAI API times out, that says
> nothing about my agent's quality — blocking a deploy on it means the gate cries wolf, and a gate
> people learn to ignore is worse than no gate. So `_error_result` returns `layer2_passed: True` with
> the error in `warnings`. The safety net is that Layer 1 still gates hard and never calls an external
> API, so structural failures always block. The caveat is real though: if the judge broke persistently
> and nobody read the warnings, you'd silently lose semantic gating. Production-grade would alert on
> judge error *rate*, not just record it.

**Q19 — How do you know the judge is right?**
> I don't, rigorously, and that's the biggest gap in what I built. I constrained it as much as I could
> — temperature zero, forced JSON schema, required literal substring evidence from the code, explicit
> anchors for what 1.0 versus 0.7 versus 0.3 means, and a validator that rejects malformed verdicts.
> Shadow mode also let me run DeepEval alongside for comparison without trusting it to gate. But none
> of that is calibration against ground truth. The right thing is to hand-label a hundred outputs with
> expert review and measure agreement — Cohen's kappa — and publish that number alongside the scores.
> Until you do, the judge is an uncalibrated instrument, and the honest framing is that it's useful
> for detecting *change* rather than for certifying *correctness*.

**Q20 — Agent broke or test broke?**
> Layer by layer. A Layer 1 failure is unambiguous — it's a structural fact about the code, so the
> agent broke. Layer 2 is where the ambiguity lives, so every verdict carries a rationale and a
> literal evidence quote, and the report includes the full conversation transcript and the generated
> code. So I read the transcript first: if the simulator went off-script or the agent asked something
> reasonable that the persona answered badly, that's a test problem. If the transcript looks like a
> normal user interaction and the code is wrong, that's the agent. The `stop_reason` field helps a lot
> too — `simulator_error` or `max_turns_reached` points at the harness, `agent_proposed_files` with a
> bad verdict points at the agent.

---

## 11. Follow-up Questions

- "Your simulator is gpt-4o at temperature 0. What happens when OpenAI deprecates that model?"
- "Twenty cases against fifteen KB topics — what's your coverage?"
- "`aggregate_tokens` is always zero in your reports. Why?" *(Real gap — the WS client ignores the
  `token_usage` event. Own it.)*
- "How do you stop the baseline from being poisoned by a bad run that nobody reviewed?"
- "What if the agent passes all 20 cases and is still bad?"
- "Your judge is gpt-4o-mini and the agent is gpt-5.3-codex. Is a weaker model qualified to grade a
  stronger one?"
- "You run k repetitions serially. What's the wall-clock for k=3?"
- "How would you test the harness itself?"

---

## 12. Senior Engineer Perspective

**They will care most about the epistemics.** The central question in LLM evaluation is "how do you
know your evaluator is correct," and the mature answer is that you *don't* fully, so you design for
detecting change rather than certifying correctness, and you disclose the limitation. This suite does
exactly that — baseline diffing, not absolute grading. Say that explicitly; it's the insight.

**They will notice the layering discipline.** Cheap and deterministic first, expensive and fuzzy
second, cross-case third. That's the same instinct as putting type checks before integration tests.
Layer 1 short-circuits before you pay for a judge call.

**They will probe the fail-open decision.** Have the tradeoff and the caveat ready.

**They will respect shadow mode.** Running a candidate evaluator in parallel without letting it gate
is the same pattern as shadow-traffic testing a service rewrite. It signals someone who has changed
critical infrastructure safely before.

**They will find the gaps** — no judge calibration, no CI wiring, zero token accounting, threshold by
intuition, serial execution. Name them first. A candidate who volunteers "here are the four things
that make this not yet production-grade" is dramatically more credible than one who has to be
cornered into each admission.

---

## 13. Common Mistakes

- **"We wrote tests for the AI."** Says nothing. Lead with why conventional testing fails.
- **Not explaining over-correction.** It's the most novel idea you have. If you only say "I have
  boundary tests," you've buried it.
- **Claiming the judge is accurate.** You cannot support that. Reframe to change-detection.
- **Ignoring cost and time.** 20 cases × 60s × k reps × real tokens is a real constraint that shaped
  the design (hence layer ordering and the `[DONE]` sentinel). Mentioning it shows you thought about
  operability.
- **Overstating CI integration.** There's a pytest plugin and a CLI. There is no evidence of an
  automated pipeline running this on every PR. Say "it's runnable in CI via the pytest plugin; wiring
  it into the deploy gate was the next step."
- **Forgetting the three-state verdict.** `flaky` is a design decision, not an implementation detail.

---

## 14. Cheat Sheet

**Shape:** 20 cases = 5 positive + 12 probes (6 pairs) + 3 QC. Standalone repo, network-only.

**Loop:** create project (REST) → WS auth → simulator ↔ agent multi-turn → stop on
`agent_proposes_files` | `[DONE]` | `max_turns` | error → collect `proposed_files["backtest.py"]` +
transcript.

**Layers:**
- L1 `ast.parse` predicates — deterministic, free, **hard gate**
- L2 gpt-4o-mini judge, T=0, JSON mode — 4 dims, `required_pass` gates / `advisory` warns, **fails open**
- L3 boundary pairs — A does A ∧ ¬B; B does B ∧ ¬A → `crossover_clean`

**Dims (V2):** `intent_capture`, `sdk_correctness`, `exit_completeness`, `conversation_quality`
**Thresholds:** pass ≥ 0.70 · fail < 0.50 · else unsure · regression Δ ≥ 0.15

**Voting:** k reps, `half = (n+1)//2` → pass / fail / **flaky**

**Baseline:** `baselines/last_green.json`, saved unconditionally (last *state*).
Blocking = result flip to fail, or required-dim drop ≥ 0.15. Advisory = pass → unsure/fail.

**Judge modes:** `openai` | `deepeval` | `shadow` (openai gates, deepeval compares) | `off`

**Gaps to volunteer:** no judge calibration · no CI gate · `aggregate_tokens` always 0 · 0.15 is
intuition · serial execution.
