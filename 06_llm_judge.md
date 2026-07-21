# LLM-as-a-Judge (Layer 2)

---

## 1. Interview Answer (2–3 minutes)

> Layer 2 is where I evaluate the things AST assertions can't reach — did the agent actually build
> the strategy the trader asked for, did it use the SDK idiomatically, is there an exit for every
> entry, and did it steer the conversation well. Those are semantic judgments, so I used an LLM as
> the evaluator.
>
> The design problem with LLM judges is that they're nondeterministic graders of nondeterministic
> output, which sounds like it should be useless. So most of the work went into constraining the
> judge until its output was stable enough to be a signal. Temperature zero. Forced JSON via
> `response_format={"type":"json_object"}`. An explicit anchored rubric — 1.0 means fully satisfied
> with clear evidence, 0.7 is a borderline pass, 0.3 is mostly missing — because without anchors,
> scores drift between runs. And every dimension has to return a literal substring quote from the
> code or the transcript as evidence, with an instruction to never invent evidence.
>
> Then I validate the response structurally: every expected dimension present, every one a dict with
> verdict, rationale, and evidence, verdict in the allowed set. And I re-derive the verdict from the
> score rather than trusting the model's own label — if it says 0.85 and "fail," the score wins.
> That's an important detail, because the model is much better at producing a calibrated number than
> at consistently applying a threshold to it.
>
> The other design decision I'd highlight is shadow mode. I had two judge backends — my own
> gpt-4o-mini judge and DeepEval's G-Eval. Shadow mode runs both, but only mine gates pass/fail; the
> DeepEval verdict rides along in the result as `shadow_verdict` purely for comparison. That's how you
> evaluate a replacement evaluator without trusting it first — same idea as shadow traffic when you
> rewrite a service.
>
> And the honest caveat I'd give you: I never calibrated the judge against human expert labels. So I
> treat it as an instrument for detecting *change* against a baseline, not for certifying
> correctness.

---

## 2. What Problem Were We Solving?

**Business.** The failures that actually hurt users are semantic, not structural. Code that parses,
subclasses `Strategy`, and implements the *wrong strategy* is far more dangerous than code that
crashes — because it produces plausible numbers a trader might act on.

**Technical.** How do you write `assert` for "does this implement a 20/50 SMA crossover with a 2%
stop"? You can't enumerate valid implementations; there are infinitely many. You need a grader that
understands intent.

**Why existing approaches weren't enough:**
- **String/AST matching** — can't express intent. Covered in [05](05_ast_validation.md).
- **Human review** — accurate but unscalable and non-reproducible. Nobody hand-reviews 20 outputs
  before a one-line prompt fix.
- **Execution-based scoring** (run the backtest, check the Sharpe) — expensive, slow, needs market
  data, and a strategy can be *correctly implemented* and unprofitable. Profitability is not
  correctness.
- **Generic eval frameworks alone** — DeepEval and Ragas score text quality generically. They don't
  know that `ExitRule` is correct for a fixed-percentage stop but wrong for a trailing one. That's
  why the primary judge is a custom prompt encoding SDK rules, with DeepEval available as a
  comparison backend rather than the gate.

---

## 3. Architecture

```
transcript (list[{role, content}])  +  intent_spec  +  generated code
                          │
                          ▼
        ┌─────────────────────────────────────────────┐
        │ layer2.py :: run_layer2(judge_mode=...)     │
        └────┬──────────┬──────────┬──────────┬───────┘
             │          │          │          │
          "off"    "openai"   "deepeval"   "shadow"
             │          │          │          │
             │          │          │     ┌────┴─────┐
             │          │          │     ▼          ▼
             │          │          │  openai    deepeval
             │          │          │  (GATES)   (rides along
             │          │          │             as shadow_verdict)
             ▼          ▼          ▼
      null verdict  judge/client.py    deepeval_client.py
      layer2_passed     │                  G-Eval
        = True          ▼
                 build_conversation_message()
                   ├─ full transcript, turn-numbered
                   ├─ intent_spec JSON (must_have / must_not_have)
                   └─ generated backtest.py
                        │
                        ▼
                 gpt-4o-mini
                   temperature = 0.0
                   max_tokens  = 2048
                   response_format = json_object
                        │
                        ▼
                 _validate_verdict()   ← all dims present? shape ok?
                 _normalize_scores()   ← clamp [0,1]; RE-DERIVE verdict
                        │
                        ▼
        ┌───────────────────────────────────────────┐
        │ _build_result()                           │
        │   dim in required_pass + not "pass" → FAIL│
        │   dim in advisory      + not "pass" → warn│
        │   waived dims → suppressed entirely       │
        │   layer2_passed = (failures == 0)         │
        └───────────────────────────────────────────┘
                        │
                        ▼
              baseline diff (Δ ≥ 0.15 on required = blocking)
```

**On judge failure** (`_error_result`): every dimension → `unsure`/0.5, `layer2_passed: True`,
error recorded in `warnings`. **Fails open by design.**

---

## 4. Deep Technical Explanation

### The four dimensions (V2)

| Dimension | Question | Evaluated against |
|---|---|---|
| `intent_capture` | Does the code reflect everything the trader expressed across the whole conversation? | transcript + `intent_spec` + code |
| `sdk_correctness` | Are SDK primitives used correctly? | code |
| `exit_completeness` | Is there an exit path for every entry path? | code |
| `conversation_quality` | Did the agent ask the right questions, at the right time, without redundancy? | **transcript only** |

There's a V1 set (`intent_fidelity`, `sdk_correctness`, `exit_completeness`) kept for back-compat.
The V1→V2 evolution is itself worth mentioning: `intent_fidelity` scored code against a single
prompt; `intent_capture` scores it against an entire multi-turn conversation. That change came from
moving to multi-turn testing — once the user simulator existed, "did it match the prompt" was the
wrong question, because the requirements emerge across turns.

`conversation_quality` is explicitly marked "Score against the transcript, NOT the code. This is
advisory." *Why advisory?* It's the most subjective dimension and the most prone to judge opinion.
Letting it gate would produce failures nobody agrees with, and a gate people disagree with is a gate
they route around.

### `intent_spec` — making intent checkable

Every case YAML declares structured intent:

```yaml
intent_spec:
  strategy_type: sma_golden_cross
  asset_class: indian_equity_index
  timeframe: 1h
  must_have:
    - "20-period SMA"
    - "2% stop-loss"
    - "sma.config() in __init__ for declarative SDK indicator registration"
  must_not_have:
    - "short entries"
```

The judge prompt then makes scoring mechanical rather than vibes-based:

> Every item in `intent_spec.must_have` appears in the code (score down per missing item).
> No item in `intent_spec.must_not_have` appears (**score 0 if any present**).

*Why this matters:* it converts an open-ended "is this right?" into a checklist with a hard zero for
violations. That's the single biggest lever on judge consistency — you're not asking the model to
have taste, you're asking it to verify enumerated claims. The `must_not_have` hard-zero is what makes
the boundary probes work at Layer 2 as well as Layer 3.

### Prompt design — the constraints that make it stable

**Anchored scale.** The system prompt enumerates what each score means:

```
1.00 — fully satisfied, clear evidence in the code
0.85 — satisfied with minor gaps
0.70 — borderline pass (default for "good enough but not great")
0.50 — partially satisfied; significant gaps
0.30 — mostly missing; weak evidence at best
0.00 — absent or contradicted
```

Without anchors, "score this 0–1" drifts run to run and between models. With them, scores cluster on
the anchors and become comparable across runs — which is the entire premise of baseline diffing.

**Mandatory literal evidence.** "Cite literal substring quotes from the code." Two effects: it's a
weak grounding constraint that suppresses hallucinated justification, and it makes every score
auditable by a human reading the report.

**Explicit anti-hallucination instruction.** "If evidence is absent, score low — never invent
evidence." Naming the failure mode in the prompt measurably reduces it.

**Role separation.** Code-side dimensions cite code; conversation-side dimensions cite literal
user/assistant turns. Prevents the judge from justifying a code score with something the user merely
*said*.

**Turn-numbered transcript.** `_format_conversation` renders `[Turn 3 — assistant]` so the judge can
reason about ordering — essential for "did it generate too early?"

### Validation and score normalization

```python
def _validate_verdict(verdict, *, expected_dims):
    # every dim present; each a dict; verdict/rationale/evidence fields exist;
    # verdict ∈ {"pass","fail","unsure"}   else raise
```

Then, critically:

```python
def _normalize_scores(verdict, *, dims):
    score = clamp(float(score), 0.0, 1.0)      # or map from verdict if absent
    d["verdict"] = derive_verdict(d["score"])   # ← score is authoritative
```

**The verdict is re-derived from the score, always.** If the model returns `score: 0.85, verdict:
"fail"`, the score wins and the verdict becomes `pass`. *Why?* Because thresholds are a mechanical
rule (`≥0.7 pass`, `<0.5 fail`) and models apply mechanical rules inconsistently while producing
reasonably calibrated numbers. Never let the model do arithmetic you can do yourself. This is a small
piece of code and a genuinely good instinct to be able to explain.

There's also a fallback map (`pass→0.85, unsure→0.6, fail→0.3`) for when a score is missing entirely,
so a partial response still yields something usable.

### Shadow mode — migrating an evaluator safely

```python
if judge_mode == "shadow":
    primary_verdict = _run_openai()      # GATES
    shadow_verdict  = _run_deepeval()    # comparison only
    result["shadow_verdict"] = shadow_verdict
    result["shadow_backend"] = "deepeval"
```

Note the shadow path has its own exception handling: a DeepEval failure fills in 0.5/`unsure`
placeholders and records `shadow_error` — **it cannot fail the run**. The shadow must never affect
the primary.

*Why this exists:* replacing your evaluator is dangerous, because you have no way to evaluate your
evaluator. Shadow mode gives you paired verdicts on identical inputs over many runs, so you can
measure agreement before switching. It's exactly shadow-traffic testing applied to evaluation
infrastructure, and it's the most sophisticated design decision in this layer.

### Gating policy — required, advisory, waived

Three tiers per case:

```yaml
judge:
  required_pass: [intent_capture, sdk_correctness]   # fail → case fails
  advisory: [exit_completeness, conversation_quality] # fail → warning only
waivers:
  - dimension: conversation_quality
    reason: "known verbosity issue, tracked in TICKET-123"
    expires: 2026-07-01
```

Required dimensions differ per case *by design* — the SMA case gates on `intent_capture` and
`exit_completeness`; the B1 probe gates on `intent_capture` and `sdk_correctness`, because the whole
point of that probe is SDK API selection.

**Waivers** suppress a dimension entirely — in `compute_diff`, a waived dimension produces no diff at
all, and removing a waiver emits an info diff showing the now-unmasked score. This lets you keep a
failing test *in the suite* with a documented reason and an expiry, instead of the usual outcome
where someone deletes the test to get to green.

### Fail-open — the deliberate risk

```python
def _error_result(dims, exc):
    return {..., "layer2_passed": True, "warnings": [f"Layer 2 judge error: {exc}"]}
```

An OpenAI timeout says nothing about agent quality. Blocking a deploy on it trains people to ignore
the gate. Layer 1 still hard-gates and never calls an external API.

**The caveat you must state:** if the judge broke persistently and nobody read warnings, semantic
gating silently disappears. Production-grade would alert on judge *error rate*.

---

## 5. My Actual Contribution

**What existed** — nothing.

**What I built** — the whole judge subsystem: `judge/prompt.py` (system prompt, anchored rubric,
V1 and V2 templates, transcript and intent formatting), `judge/client.py` (OpenAI call, retry,
validation, score normalization, null verdicts), `judge/schema.py` (dimension sets, thresholds,
`derive_verdict`), `judge/deepeval_client.py` (G-Eval backend), and `assertions/layer2.py` (mode
dispatch, shadow logic, required/advisory/waiver gating, fail-open error handling).

**What changed** — semantic quality became a number that could be tracked across runs and diffed
against a baseline.

**Result** — four-dimension scoring with mandatory evidence, per-case gating policy, a safe migration
path for the evaluator itself, and integration with baseline regression detection at a 0.15 threshold.

---

## 6. Files and Code

| Path | Contents |
|---|---|
| `harness/judge/prompt.py` | `SYSTEM_PROMPT`, `USER_TEMPLATE` (V1), `CONVERSATION_TEMPLATE` (V2), `build_user_message`, `build_conversation_message`, `_format_conversation`, `_format_intent_spec` |
| `harness/judge/client.py` | `_call_openai`, `run_judge`, `run_judge_conversation`, `_validate_verdict`, `_normalize_scores`, `make_null_verdict`, `JudgeError` |
| `harness/judge/schema.py` | `ALL_DIMENSIONS`, `V1_DIMENSIONS`, `V2_DIMENSIONS`, `VALID_VERDICTS`, `SCORE_PASS_THRESHOLD=0.7`, `SCORE_FAIL_THRESHOLD=0.5`, `SCORE_REGRESSION_THRESHOLD=0.15`, `derive_verdict` |
| `harness/judge/deepeval_client.py` | `run_deepeval_conversation` — G-Eval backend |
| `assertions/layer2.py` | `run_layer2`, `_build_result`, `_error_result` |
| `harness/baseline.py` | `extract_score`, `compute_diff` — consumes judge scores |
| `harness/scorecard.py` | Per-dimension aggregation, weakest-case identification |

**Model:** `gpt-4o-mini`, `temperature=0.0`, `max_tokens=2048`,
`response_format={"type": "json_object"}`, `max_retries=1`.

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| LLM judge at all | Only way to score intent at scale | Nondeterministic; uncalibrated |
| `gpt-4o-mini` not a frontier model | 20 cases × k reps × 4 dims — cost adds up | Weaker judge grading a stronger generator |
| Temperature 0.0 | Maximize reproducibility | Doesn't eliminate variance |
| Forced JSON mode | Parse reliability | Costs a little instruction-following headroom |
| Anchored 0–1 rubric | Makes scores comparable across runs | Anchors are still judgment calls |
| Mandatory literal evidence | Grounding + human auditability | Consumes output tokens |
| Re-derive verdict from score | Models apply thresholds inconsistently | None meaningful |
| Custom prompt over generic framework | Encodes SDK-specific rules | Maintenance as the SDK evolves |
| DeepEval as shadow, not gate | Safe evaluator migration | Doubles judge cost in shadow mode |
| Required vs advisory per case | Different cases test different things | Per-case configuration burden |
| Waivers with expiry | Keep failing tests in the suite | Needs enforcement or they rot |
| Fail open on judge error | Flaky API must not block deploys | Silent loss of gating if unnoticed |
| `max_retries=1` | One retry on malformed JSON | Two total attempts may not be enough |

---

## 8. Production Improvements

1. **Calibrate against human labels.** The single biggest gap. 100 hand-labeled outputs, measure
   Cohen's κ, publish it with every report. Without it the judge is an uncalibrated instrument.
2. **Measure judge variance directly.** Run the same input N times, compute σ per dimension. That
   both quantifies noise and gives a principled basis for the 0.15 regression threshold.
3. **Self-consistency sampling.** Three judge calls, take the median score. Standard, effective,
   triples cost — worth it on required dimensions only.
4. **Position/verbosity bias testing.** LLM judges reliably favour longer outputs and are sensitive to
   ordering. Test with padded and reordered inputs.
5. **Alert on judge error rate**, not just record it — closes the fail-open hole.
6. **Ensemble across model families** for required dimensions; disagreement flags a case for human
   review.
7. **Enforce waiver expiry** in CI.
8. **Cache judge verdicts** keyed on `hash(code + transcript + prompt_version)` — reruns are free.
9. **Version the judge prompt** and record the version in every report. A prompt change today
   invalidates baseline comparability silently.
10. **Track score distributions over time**, not just pairwise diffs — reveals slow erosion.

---

## 9. Common Interview Questions

1. What is LLM-as-a-judge and why use it?
2. How do you trust a nondeterministic grader?
3. What are your dimensions and why those?
4. Why is `conversation_quality` advisory?
5. How does `intent_spec` improve judging?
6. Why an anchored rubric?
7. Why require literal evidence quotes?
8. Why re-derive the verdict from the score?
9. What's shadow mode and why does it exist?
10. Why gpt-4o-mini and not a frontier model?
11. Is a weaker model qualified to grade a stronger one?
12. What are the hallucination risks?
13. What happens when the judge API fails?
14. Isn't fail-open dangerous?
15. What's a waiver?
16. How do you detect judge drift over time?
17. How do you know the judge is accurate?
18. What biases do LLM judges have?
19. Why temperature 0 — does it make it deterministic?
20. How would you replace the judge with something better?

---

## 10. Strong Answers

**Q2 — Trusting a nondeterministic grader.**
> I mostly don't, and the architecture reflects that. I constrain it hard — temperature zero, forced
> JSON, an anchored rubric so scores mean the same thing across runs, mandatory literal evidence
> quotes, structural validation of the response, and re-deriving verdicts from scores rather than
> trusting the model's own labels. That gets variance low enough to be useful. But the real answer is
> that I don't use it as an absolute grader. I use it as a *change detector*: every run diffs against
> a stored baseline, and what fails the build is a required dimension dropping more than 0.15 from its
> previous score. Systematic judge bias mostly cancels in a diff — if the judge is consistently
> generous, it's generous on both sides of the comparison. That reframing is what makes an
> uncalibrated instrument genuinely useful.

**Q11 — Weaker model grading a stronger one?**
> It's a fair challenge and I'd concede the concern. My defence is that judging is an easier task than
> generating. Writing a correct pairs-trading strategy from a conversation requires synthesis;
> checking whether a named item from `must_have` appears in the code is closer to verification, and I
> deliberately structured the prompt to make it verification — the `intent_spec` turns an open-ended
> quality judgment into a checklist. That said, `sdk_correctness` is the dimension where I'd trust it
> least, because it requires genuine SDK understanding. If I were hardening this I'd run required
> dimensions through a stronger model, or ensemble two families and flag disagreements for human
> review, and I'd keep the cheap model for the advisory dimensions.

**Q12 — Hallucination risks.**
> Three distinct ones. The judge can invent evidence — cite a line that isn't in the code — which I
> attack by requiring literal substring quotes and explicitly instructing it never to invent evidence,
> though nothing programmatically verifies the quote actually appears in the source. That's a real gap
> and a genuinely easy fix: substring-check the evidence field and downgrade the verdict if it fails.
> Second, it can rationalize a score after the fact, producing a plausible rationale for a number it
> picked for other reasons — the anchored rubric constrains this but doesn't eliminate it. Third, and
> subtlest, it can be fooled by *code that looks right*: well-named variables and clean structure bias
> it toward higher `sdk_correctness` scores than the logic deserves. That last one is why Layer 1's
> deterministic checks exist underneath and why `must_not_have` carries a hard zero.

**Q18 — Judge biases.**
> The documented ones are verbosity bias — longer answers score higher regardless of quality —
> position bias in pairwise comparisons, self-preference where a model favours output from its own
> family, and anchoring on the first dimension scored. I mitigated some structurally: scoring
> dimensions independently rather than comparatively avoids position bias entirely, and the anchored
> rubric limits drift. I did not test for verbosity bias, which I'd flag as a real gap — our agent
> writes Jupytext files with mandatory markdown cells, so outputs are verbose by construction, and
> that's exactly the condition where verbosity bias would inflate scores uniformly. The saving grace
> is again the baseline diff: a uniform inflation affects both sides of the comparison.

**Q19 — Does temperature 0 make it deterministic?**
> No, and it's worth being precise about that. Temperature zero makes sampling greedy, but you still
> get run-to-run variation from batching and floating-point non-associativity on GPU inference, and
> from silent model updates behind a versioned endpoint. In practice I saw scores that were stable to
> roughly the anchor granularity but not bit-identical. That's precisely why the harness runs k
> repetitions with a majority vote and reports a `flaky` verdict on a split, and why the regression
> threshold is 0.15 rather than anything tighter — the threshold has to sit above the noise floor. If
> I'd measured that noise floor properly I could have set it from data rather than intuition.

---

## 11. Follow-up Questions

- "You require literal evidence quotes. Do you verify the quote actually appears in the source?"
  *(No — and it's a ten-line fix. Say so.)*
- "Your judge prompt is unversioned. What happens to your baselines when you edit it?"
- "0.7 pass, 0.5 fail — where did those come from?"
- "In shadow mode, what would make you switch to DeepEval?"
- "A case has `required_pass: [intent_capture]` and `must_not_have` is violated. Walk me through the
  scoring."
- "What if the judge and Layer 1 disagree?"
- "How much does a full run cost in judge tokens?"

---

## 12. Senior Engineer Perspective

**The epistemics are the whole conversation.** "How do you evaluate the evaluator" is the central
unsolved problem in LLM evaluation. The mature position — you can't fully, so design for change
detection, constrain aggressively, and disclose the limitation — is the answer. Candidates who claim
their judge is accurate reveal they haven't thought about it.

**Shadow mode will land well.** It shows someone who has swapped critical infrastructure without
breaking it, and who understands that an evaluator is itself a system requiring validation.

**They'll probe the fail-open decision** and want to hear both the reasoning and the residual risk.

**They'll ask about cost.** 20 cases × k reps × judge calls is real spend, and it shaped the design:
cheap model, layer ordering so Layer 1 short-circuits, `off` mode for fast iteration.

**They'll find the calibration gap.** Volunteer it before they ask. "The judge is uncalibrated against
human labels, so I use it for detecting change rather than certifying correctness" is a sentence that
buys you enormous credibility.

---

## 13. Common Mistakes

- **Claiming judge accuracy.** You have no ground-truth comparison. Reframe to change detection.
- **"We used GPT to check the output."** Says nothing about the engineering. The rubric anchoring,
  evidence requirement, and verdict re-derivation are the substance.
- **Not knowing the thresholds.** 0.7 / 0.5 / 0.15. Memorize them.
- **Forgetting fail-open.** It's a design decision with a named risk, not an oversight.
- **Ignoring bias.** Every interviewer who has worked with LLM judges knows about verbosity bias.
  Raising it yourself signals real experience.
- **Presenting DeepEval as the primary.** It's the shadow. The custom judge gates, because generic
  frameworks don't know SDK semantics.

---

## 14. Cheat Sheet

- **Model:** `gpt-4o-mini` · T=0.0 · `max_tokens=2048` · `response_format=json_object` · 1 retry.
- **V2 dims:** `intent_capture` · `sdk_correctness` · `exit_completeness` · `conversation_quality`
  (advisory, transcript-only).
- **V1 (legacy):** `intent_fidelity` · `sdk_correctness` · `exit_completeness`.
- **Thresholds:** pass ≥ **0.70** · fail < **0.50** · else `unsure` · regression Δ ≥ **0.15**.
- **Anchors:** 1.00 full · 0.85 minor gaps · 0.70 borderline · 0.50 partial · 0.30 mostly missing ·
  0.00 absent.
- **`intent_spec`:** `must_have` (score down per miss) · `must_not_have` (**hard zero** if present).
- **Modes:** `openai` (default gate) · `deepeval` (G-Eval) · `shadow` (openai gates, deepeval
  compares) · `off`.
- **Gating tiers:** `required_pass` fails the case · `advisory` warns · `waivers` suppress with reason
  + expiry.
- **Key mechanic:** verdict is **re-derived from score** — never trust the model's own label.
- **Fail open:** judge error → `layer2_passed: True` + warning. Layer 1 still hard-gates.
- **Admit:** no human calibration · evidence quotes unverified · prompt unversioned · verbosity bias
  untested.
