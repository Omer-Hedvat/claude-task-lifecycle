# AI Engineering Layer — learned router + evals + telemetry

| Field | Value |
|---|---|
| **Type** | Epic |
| **Phase** | AI Engineering |
| **Status** | `not-started` |
| **Tracker** | `TOOLING_ROADMAP.md` |
| **Children** | 3 tasks |
| **Rollup** | 0/3 wrapped · 3 not-started |
| **Depends on** | task lifecycle substrate (`TASK_MGMT_HARDENING_SPEC.md` status model + telemetry hook points) |

---

## Overview

This Epic is what turns `claude-task-lifecycle` from a developer-tooling workflow into a
**measured agentic system**. It adds the three components that production LLM systems are
actually judged on — a learned routing model, an evaluation harness, and operational
telemetry/analysis — on top of the existing task lifecycle.

It is deliberately scoped so each child is independently demonstrable and produces a
publishable artifact (a metric, a curve, a writeup), not just code.

---

## Child 1 — `rightsizer_router` (the learned model router)

**Goal.** Given a task's features, predict the cheapest model tier + effort that will still
pass QA.

- **Inputs (features):** diff size / files touched, task type (bug/feature/chore/epic),
  declared domain, historical failure rate for similar tasks, spec length.
- **Output:** `{ model_tier: haiku|sonnet|opus, effort: low|med|high, qa_tier: 1..N }`.
- **v0 (rules + thresholds):** a transparent heuristic baseline — the control to beat.
- **v1 (learned):** a lightweight classifier trained on logged `(features → QA outcome)`
  pairs from telemetry; falls back to v0 when confidence is low.
- **Deliverable:** a **cost-vs-quality tradeoff curve** — $/task and latency per tier vs.
  QA pass-rate retained. Show that routing saves $ without losing quality.

## Child 2 — `eval_harness`

**Goal.** Make every routing/lifecycle change measurable; catch regressions before they ship.

- **Golden set:** a curated set of representative tasks with known-good outcomes.
- **Scoring:** deterministic checks (build/test/grep assertions) + **LLM-as-judge** for
  open-ended steps, with the judge prompt + rubric version-controlled.
- **Metrics:** task-completion rate, regression catches, tool-call accuracy, and
  **router tier-selection accuracy** (did the router pick the cheapest tier that passed?).
- **Gate:** CI runs the eval suite; a routing change must not regress the metrics.
- **Deliverable:** a published before/after metrics table in the README.
- **Detailed design:** `docs/eval-harness-design.md` (metrics, golden-set schema, judge
  calibration, router oracle, CI gate, phased build plan E0–E4).

## Child 3 — `telemetry_analysis`

**Goal.** Instrument runs and close the learning loop.

- **Instrumentation:** per-task tokens, latency, tier chosen, QA outcome, retries, cost.
  Opt-in only, documented off-switch, no content captured by default.
- **Analysis:** which task features predict QA failure? where does the router mis-size?
  A short **writeup** (notebook or markdown) with charts — the genuine data-analysis artifact.
- **Loop:** the cleaned `(features → outcome)` table is the training input for Child 1 v1.

---

## Invariants

- No metric is claimed without the eval/telemetry behind it (honesty over hype).
- The router must have a transparent rules baseline (v0) so the learned model's lift is
  measurable, not assumed.
- All thresholds/weights are config or learned — never hardcoded per-project literals.
- Telemetry is opt-in and privacy-preserving by default.

## Portfolio note

This Epic is the AI-Engineering centerpiece of the project. Each child maps to a named
2026 AI-engineering screening signal: **model routing / cost optimization** (Child 1),
**eval design** (Child 2), **observability + data analysis** (Child 3).
