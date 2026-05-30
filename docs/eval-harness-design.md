# Eval Harness — Design

Build-ready design for the `eval_harness` child of the AI Engineering Layer epic
(`future_devs/AI_ENGINEERING_LAYER_SPEC.md`). Companion to the `rightsizer_router` and
`telemetry_analysis` children — the harness is the contract those two are measured against.

> One-line thesis: **no routing or lifecycle change ships without moving (or holding) the
> numbers.** The eval harness is what makes "adaptive" and "right-sized" claims falsifiable.

---

## 1. What is under test

The system under test (SUT) is the **orchestration layer**, not a single prompt. A run
takes a *task* and produces a *trajectory* + a *final repo state*:

```
task (features) ──▶ RightSizer decision {tier, effort, qa_tier}
                       ──▶ agent executes in an isolated git worktree
                              ──▶ lifecycle: file → start → QA → wrap
                                     ──▶ outcome {diff, QA result, cost, latency, retries}
```

We evaluate three distinct things, and must not conflate them:

| Layer | Question | Primary metric |
|---|---|---|
| **Task outcome** | Did the agent actually solve the task correctly? | task-completion rate |
| **Router decision** | Did `RightSizer` pick the cheapest tier that still passes? | tier-selection accuracy, cost saved @ quality held |
| **QA gating** | Did the QA tier catch real breakage and not false-flag good work? | regression-catch rate, false-positive rate |

Keeping these separable is the point: a task can fail because the *model* was too weak
(outcome) **or** because the *router* under-sized it (decision). The harness must attribute
the failure to the right layer.

---

## 2. Metrics (precise definitions)

Let a case run yield `passed ∈ {0,1}` (deterministic checks + judge agree it's correct),
`tier ∈ {haiku, sonnet, opus}`, `cost` (USD), `latency` (s).

- **Task-completion rate** = `mean(passed)` over the golden set. Report with a 95% Wilson
  confidence interval (small-N honesty).
- **Tier-selection accuracy** = `mean(chosen_tier == oracle_tier)`, where `oracle_tier` is
  the *cheapest tier that passes* for that case (see §4 for how the oracle is built).
- **Over-/under-sizing rate** = fraction where `chosen_tier > oracle_tier` (wasted money)
  vs `chosen_tier < oracle_tier` (quality risk). These trade off; report both.
- **Cost saved @ quality held** = `(cost_all_opus − cost_routed) / cost_all_opus`,
  reported *only* alongside the completion-rate delta vs the all-Opus baseline. A cost
  number without the paired quality number is meaningless and is forbidden by the honesty
  invariant.
- **Regression-catch rate** = of cases with a deliberately-broken variant, the fraction the
  QA tier flagged. **False-positive rate** = of known-good cases, the fraction QA wrongly
  flagged. (A gate that flags everything has 100% catch and is useless — both numbers ship
  together.)
- **Tool-call accuracy** = fraction of expected tool calls (e.g. `start_task` before edits)
  that occurred, from the trajectory.

Headline artifact = the **cost-vs-quality frontier**: completion rate (y) vs $/task (x) for
{all-Haiku, all-Sonnet, all-Opus, routed}. The routed point should sit above the linear
interpolation between the static points — that *is* the router's value, made visible.

---

## 3. The golden set

### Schema (one YAML file per case, `evals/cases/*.yaml`)

```yaml
id: bug-null-deref-001
category: bug            # bug | feature | chore | refactor | docs | epic-child
difficulty: low          # low | medium | high  (author estimate; oracle_tier is ground truth)
domain: viewmodel        # free-text project domain tag
fixture: repos/kotlin-mini@a1b2c3   # pinned fixture repo + commit (the worktree base)
prompt: |
  The set-completion counter double-increments when a superset is logged. Fix it.
features:                # what RightSizer sees; recorded so router evals are reproducible
  diff_size_hint: small
  files_touched_hint: 1
checks:                  # deterministic, ordered; all must pass
  - type: command
    run: "./gradlew test --tests *SupersetCounterTest*"
    expect_exit: 0
  - type: grep
    pattern: "counter\\+\\+"
    path: "src/.../SetLogger.kt"
    expect: absent
judge:                   # optional LLM-as-judge for open-ended quality
  rubric: rubrics/code-fix.md
  threshold: 0.7
broken_variant:          # optional: a seeded-bug version to test QA catch-rate
  patch: variants/bug-null-deref-001.broken.patch
```

### Composition rules

- **Size:** start at 25–40 cases; enough for signal, small enough to author well. Quality
  over quantity — every case must have a crisp, machine-checkable definition of "correct."
- **Stratify** across `category × difficulty` so per-stratum rates are reportable (the
  router is judged per-stratum, not just in aggregate).
- **Hold-out split.** Tag ~30% as `holdout: true`. Tune thresholds/rubrics on the dev split
  only; report headline numbers on holdout. Never tune on holdout — that's the contamination
  trap that makes eval numbers lie.
- **Fixtures are pinned.** Each case runs against a small fixture repo at a fixed commit, so
  results are reproducible and a real `gradlew/pytest/vitest` can run inside the worktree.
- **Provenance:** prefer cases mined from *real* closed tasks (PowerME's wrapped bugs) over
  synthetic ones — real distribution beats invented difficulty.

---

## 4. Scoring

### 4a. Deterministic checks (the spine)

Ordered `checks` run in the task's worktree: `command` (exit code / output regex),
`grep` (presence/absence), `file_exists`, `json_path`. Deterministic checks are the
**primary** pass signal because they're cheap, stable, and non-gameable. A case with no
meaningful deterministic check is a smell — fix the case, don't lean on the judge.

### 4b. LLM-as-judge (for what code-checks can't capture)

Used for open-ended quality (clarity of a refactor, whether a fix addresses root cause vs
masks a symptom). Treated as a measured component, not a black box:

- **Rubric is versioned** (`rubrics/*.md`), scored 0–1 against explicit criteria; the judge
  must return a structured `{score, reasoning, criteria_hits[]}`.
- **Prefer pairwise** (candidate vs reference solution) over pointwise — pairwise is far more
  stable and less prone to absolute-scale drift.
- **Bias mitigations:** randomize candidate position (counter position bias); strip model
  identity from the trajectory before judging (counter self-preference); cap reasoning length
  in the prompt (counter verbosity bias); use a *different model family* as judge than the one
  under test where possible.
- **Calibration is mandatory.** Hand-label ~20 cases; report judge-vs-human agreement
  (Cohen's κ). If κ < 0.6 the rubric is broken — fix the rubric, don't trust the judge. This
  calibration number goes in the report; it's the credibility anchor for every judge-based metric.

### 4c. The router oracle (how `oracle_tier` is computed)

To know whether the router right-sized, we need ground truth for "cheapest tier that
passes." Build it once per golden-set version, offline:

1. Run every case at **every** tier (Haiku, Sonnet, Opus), each `k=3` times (non-determinism).
2. A tier "passes" a case if it passes deterministic checks + judge in ≥⌈k/2⌉ runs (pass@majority).
3. `oracle_tier` = the cheapest passing tier (or `none` if no tier passes — those cases test
   the floor, not the router, and are excluded from tier-selection accuracy).

This grid is expensive (cases × 3 tiers × k), so it's a **cached artifact**
(`evals/oracle/<goldenset-sha>.json`), recomputed only when the golden set changes. Router
evals then read the cache — cheap and reproducible.

---

## 5. Non-determinism & reproducibility

- **pass@majority over k runs** (k=3 default) everywhere; report run-to-run variance, never a
  single-shot number.
- **Cassettes:** record model responses per `(case, tier, run)` to `evals/cassettes/`. CI
  replays cassettes by default (free, deterministic, fast); a nightly/manual job runs live to
  refresh them. This keeps PR CI from costing money or flaking on the model API.
- **Seeded ordering**; fixtures pinned by commit; judge model + rubric pinned by version in
  the report header so any number is traceable to exactly what produced it.

---

## 6. Architecture & tech choice

**Decision: a thin custom Python harness on `pytest`, using `DeepEval` for the
LLM-as-judge metrics (G-Eval), not an off-the-shelf prompt-eval tool.**

Rationale (this is a judgment call worth recording): prompt-eval tools like `promptfoo` model
*prompt in → text out*. Our SUT is an *agent that mutates a git worktree and runs a build* —
the unit of truth is repo state + a trajectory, which those tools don't represent. A custom
runner that orchestrates worktree setup → task run → deterministic checks, and delegates only
the *judging* step to a proven library, fits the SUT and stays portfolio-legible (Python,
pytest, CI — the expected AI-Eng stack).

```
evals/
├── cases/*.yaml            # the golden set
├── rubrics/*.md            # versioned judge rubrics
├── fixtures/               # pinned mini-repos (or submodules) cases run against
├── oracle/<sha>.json       # cached cheapest-tier-that-passes ground truth
├── cassettes/              # recorded model responses for deterministic CI
├── runner/
│   ├── harness.py          # load case → worktree → run SUT → collect outcome
│   ├── checks.py           # deterministic check executors
│   ├── judge.py            # DeepEval/G-Eval wrapper + bias mitigations
│   ├── oracle.py           # build/refresh the tier oracle grid
│   └── metrics.py          # metric formulas + Wilson CI + frontier plot
├── reports/                # generated markdown/HTML + the frontier PNG
└── test_evals.py           # pytest entrypoint; CI gate lives here
```

---

## 7. CI gating

- Runs on every PR via cassettes (free, deterministic). Live run is nightly + manual dispatch.
- **Gate = no regression beyond noise.** Compare against a committed `evals/baseline.json`.
  Fail the PR if any headline metric drops by more than its run-to-run std-dev (so noise
  doesn't false-fail, but real regressions do). Improving a metric updates the baseline in the
  same PR (reviewer sees the delta).
- CI publishes the report + frontier plot as a build artifact and a PR comment, so every
  change shows its measured effect inline.

```yaml
# .github/workflows/evals.yml (sketch)
on: [pull_request]
jobs:
  evals:
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e ".[evals]"
      - run: pytest evals/ --replay-cassettes --baseline evals/baseline.json
      - uses: actions/upload-artifact@v4
        with: { name: eval-report, path: evals/reports/ }
```

---

## 8. Build plan (phased — each phase ships a usable artifact)

| Phase | Deliverable | Done when |
|---|---|---|
| E0 | Harness skeleton + 5 deterministic-only cases + `pytest` runner | `pytest evals/` runs 5 cases against fixtures and reports completion rate + Wilson CI |
| E1 | LLM-as-judge + rubric + human calibration (κ reported) | judge agrees with human labels at κ ≥ 0.6 on 20 cases |
| E2 | Golden set to 25–40 cases, stratified, dev/holdout split | per-stratum completion rates reported on holdout |
| E3 | Router oracle grid + tier-selection accuracy + **cost-vs-quality frontier plot** | frontier PNG generated; routed point beats static interpolation (or honest writeup if it doesn't) |
| E4 | Cassettes + CI gate + baseline + PR-comment report | a PR that regresses a metric fails CI; passing PR posts the report |

E3 is the portfolio money-shot. E0–E1 are the credibility foundation (calibrated judge >
big-but-unvalidated golden set).

---

## 9. Risks & open questions

- **Judge unreliability** — mitigated by pairwise + calibration + deterministic-first scoring;
  if κ stays low, lean harder on deterministic checks and shrink the judge's role.
- **Oracle cost** — the cases × tiers × k grid is the expensive part; bounded by keeping the
  set small and caching by golden-set SHA. Revisit if the set grows past ~50.
- **Fixture realism** — mini-repos may not exercise real difficulty; mitigated by mining cases
  from real wrapped PowerME tasks.
- **Overfitting the router to the golden set** — the holdout split is the guard; if dev/holdout
  diverge, the router learned the set, not the task distribution.
- **Open:** pointwise vs pairwise as the default judge mode (lean pairwise; confirm at E1).
- **Open:** does router v1 (learned) actually beat the v0 rules baseline by more than noise?
  That question is *only answerable* once this harness exists — which is the whole point.
