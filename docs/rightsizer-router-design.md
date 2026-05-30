# RightSizer Router — Design

Build-ready design for the `rightsizer_router` child of the AI Engineering Layer epic
(`future_devs/AI_ENGINEERING_LAYER_SPEC.md`). Reads alongside the eval harness design
(`docs/eval-harness-design.md`), which is how every claim here is measured.

> One-line thesis: **predict the cheapest model tier + effort that will still pass QA for a
> given task** — and prove the learned policy beats both an all-Opus baseline (on cost) and a
> transparent rules baseline (on accuracy), by more than noise.

---

## 1. What it decides

For each task, before execution:

```
features(task)  ──▶  RightSizer  ──▶  { model_tier, effort, qa_tier, confidence }
```

| Output | Range | Meaning |
|---|---|---|
| `model_tier` | `haiku < sonnet < opus` | which model runs the task (ordinal — order matters) |
| `effort` | `low / med / high` | reasoning/turn budget |
| `qa_tier` | `1..N` | how much QA the result gets (1 = lint only … N = full review+security) |
| `confidence` | `[0,1]` | calibrated; drives the abstain-to-baseline fallback (§6) |

`model_tier` is the headline decision and the one tied to the cost-vs-quality frontier.
`effort` and `qa_tier` are secondary heads trained the same way once the tier head works.

---

## 2. Features

All cheap to compute *before* running the task (no model call needed to route):

| Feature | Source | Why it predicts difficulty |
|---|---|---|
| `diff_size_hint` / `files_touched` | task spec / `Touches` list | bigger blast radius → harder |
| `task_category` | tracker (`bug/feature/chore/refactor/docs/epic-child`) | categories have different base difficulty |
| `domain` | spec routing table | some domains (DB migration, concurrency) are reliably harder |
| `spec_length` / `acceptance_criteria_count` | spec file | proxy for scope |
| `historical_failure_rate` | telemetry, same `(category,domain)` | the strongest learned signal once data exists |
| `dependency_count` | tracker `Depends on` | coordination cost |
| `keyword_flags` | spec text (e.g. "migration", "race", "security") | high-risk markers |

Feature extraction is deterministic and versioned (`features_version` stamped on every
logged decision) so a feature change doesn't silently corrupt the training set.

---

## 3. v0 — the rules baseline (ship this first)

A transparent, hand-written policy. It is **the control the learned model must beat**, and
the cold-start policy, and the abstain fallback. Roughly:

```
if category in {docs, chore} and files <= 2:        haiku, low,  qa1
elif keyword_flags ∩ {migration, security, race}:   opus,  high, qaN     # never under-size risk
elif diff_size_hint == large or files > 5:           opus,  med,  qa3
elif category == bug and files <= 2:                 sonnet, med, qa2
else:                                                sonnet, med, qa2     # safe default
```

Rules are config (`router/rules.yaml`), not code literals (design-for-extraction invariant).
Shipping v0 alone already produces a measurable frontier point — don't gate value on v1.

---

## 4. v1 — the learned policy

### 4a. Labels (this is the subtle part)

Ground truth for "cheapest tier that passes" comes from the **eval harness oracle grid**
(`evals/oracle/<sha>.json`) — every case run at every tier, `oracle_tier` = cheapest passing
tier. That grid is the *unbiased, fully-observed* label source. Production telemetry alone is
**not** sufficient for labels — see §7 (censored feedback).

### 4b. Model class — interpretable classifier first, not an LLM

**Decision: gradient-boosted trees (or monotonic-constrained GBM) over a small feature
vector — not an LLM-as-router.** Rationale, recorded because it's a real judgment call:

- Data is small and tabular; a heavy model overfits and an LLM call adds cost+latency to a
  step whose entire purpose is to *save* cost.
- The tiers are **ordinal** (`haiku<sonnet<opus`) and the error costs are **asymmetric**
  (under-sizing risks quality; over-sizing wastes money). A GBM with a cost-sensitive /
  ordinal loss expresses both directly. An LLM-as-router expresses neither cleanly.
- **Interpretability is a feature, not a nicety:** feature importances + per-decision reasons
  ("chose opus: migration keyword + 8 files") build trust and *are the writeup*.
- Inference is sub-millisecond and free — appropriate for a gate on every task.

LLM-as-router is kept as a benchmarked alternative in the eval, not the default, so the
choice is shown to be earned, not assumed.

### 4c. Objective

Minimize expected cost subject to holding quality: an asymmetric loss that penalizes
under-sizing (predicted tier < oracle tier) more than over-sizing, with the penalty ratio a
tunable reflecting risk appetite. Calibrate predicted-tier *probabilities* (isotonic /
Platt) so `confidence` is meaningful for the fallback in §6.

---

## 5. The decision → frontier link

The router's whole value is one plot (owned by the eval harness, fed by this component):
completion rate (y) vs $/task (x) for {all-Haiku, all-Sonnet, all-Opus, **routed-v0**,
**routed-v1**}. Success = the routed points sit **above** the interpolation between static
points — same quality, less money, or more quality per dollar. If they don't, the honest
result is "static Sonnet is good enough," and we say so. That falsifiability is the point.

---

## 6. Serving, cold-start, and the learning loop

- **Cold start:** no telemetry yet → v0 rules. The router never blocks on a model.
- **Abstain-to-baseline:** if v1 `confidence < τ`, fall back to v0. Tune `τ` so v1 only
  overrides the baseline where it's actually confident — bounds downside during early data.
- **ε-exploration:** with small probability, deliberately pick a non-greedy tier and log the
  outcome. This is what keeps the feedback data from collapsing onto the policy's own
  choices (§7) — a few % exploration buys unbiased signal over time.
- **Retrain cadence:** offline, on a schedule (e.g. weekly) or every N new labeled tasks,
  gated by the eval harness — a new model ships only if it holds/improves the metrics on the
  holdout split. No silent online weight updates.

---

## 7. Risks & failure modes

- **Censored / off-policy feedback (the big one).** Production only observes the outcome for
  the tier the router *chose* — we never see how the unchosen tiers would have done. Naively
  training on this biases toward the current policy. Mitigations, in order: (1) the offline
  oracle grid gives fully-observed labels; (2) ε-exploration adds unbiased on-policy samples;
  (3) inverse-propensity weighting if we later go full contextual-bandit. Treating routing as
  a **contextual bandit**, not plain supervised learning, is the correct framing.
- **Distribution shift.** A new project/stack shifts the feature distribution; the abstain
  fallback + per-project monitoring catch it, and `historical_failure_rate` re-learns.
- **Asymmetric-error complacency.** Aggregate accuracy can look fine while under-sizing the
  rare high-risk task. Report under-sizing rate *per stratum*, and hard-floor risky keywords
  to Opus in both v0 and v1 (safety rail outside the learned policy).
- **Feedback-loop overfitting.** Router learns the golden set, not the task distribution —
  guarded by the eval holdout split.
- **Gaming / spec drift.** Features derived from author-written spec text can be (un)wittingly
  skewed; keep deterministic features dominant and monitor importances for over-reliance on
  soft signals.

---

## 8. Architecture & layout

```
router/
├── rules.yaml             # v0 policy (config, not code)
├── features.py            # deterministic feature extraction (+ features_version)
├── policy_v0.py           # rules engine
├── policy_v1.py           # GBM: train / predict / calibrate / explain
├── bandit.py              # ε-exploration + propensity logging
├── train.py               # offline trainer; reads oracle grid + telemetry, gates on evals
└── serve.py               # route(task) -> decision; v1 with confidence-gated v0 fallback
```

Training inputs: `evals/oracle/<sha>.json` (unbiased labels) + telemetry
(`telemetry_analysis` child) for `historical_failure_rate` and exploration samples. Outputs
feed the eval harness, which produces the frontier and the CI gate.

---

## 9. Build plan (phased)

| Phase | Deliverable | Done when |
|---|---|---|
| R0 | v0 rules engine in `rules.yaml` + `serve.py` | every task gets a routed decision; produces the `routed-v0` frontier point |
| R1 | Feature extraction + decision logging (with propensities) | every decision logged with features + chosen tier + outcome |
| R2 | Train v1 GBM on the oracle grid; calibrate probabilities | tier-selection accuracy vs v0 reported on the eval holdout |
| R3 | Confidence-gated fallback + ε-exploration in serving | v1 overrides v0 only above τ; exploration samples accumulating |
| R4 | **Cost-vs-quality frontier with routed-v0 & routed-v1** + writeup | frontier plot shows routed beats static interpolation, or an honest null result is written up |

R0 ships value immediately (a real routing baseline). R2 is the first learned number. R4 is
the portfolio money-shot, shared with the eval harness's E3.

---

## 10. Open questions

- Single multi-head model vs. separate models for `tier` / `effort` / `qa_tier`? Start with
  the tier head only; add heads once it clears the baseline.
- Ordinal-regression GBM vs. multiclass with a monotonic constraint — decide empirically at R2.
- How much exploration (ε) trades short-term cost for long-term label quality — tune against
  the frontier.
- **The headline open question (shared with the eval design):** does v1 beat v0 by more than
  run-to-run noise? Unanswerable until R2 + the eval harness exist — which is exactly why both
  are designed before either is built.
