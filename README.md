# claude-task-lifecycle

> An **adaptive agent-orchestration layer** for AI coding agents: a git-native task
> lifecycle with a *learned model router* and *eval-driven QA gating*, designed to run
> across multiple projects.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
![Status](https://img.shields.io/badge/status-incubating-orange.svg)

---

## Why this exists

I run AI coding agents across several real repositories (an Android fitness app, a
data-science workspace, a web project). Driving them well is not a prompting problem —
it's an **orchestration** problem: which task runs next, on which model tier, with how
much QA, isolated from which other work, and at what cost. I built a workflow to answer
that for my own projects; this repo is the effort to generalize it into a reusable,
**measured** system.

The interesting engineering is not the task list. It's three things that map directly
to how production agentic systems are actually built and evaluated:

1. **A learned model router (`RightSizer`)** — picks the model tier (Haiku / Sonnet /
   Opus) and effort per task from task features (diff size, file count, domain, risk),
   and *learns from QA pass/fail outcomes* which tier was actually sufficient. The goal
   is a published **cost-vs-quality tradeoff curve** ($/task per tier, quality retained).
2. **An eval harness** — a golden set of representative tasks scored with deterministic
   checks + LLM-as-judge, producing metrics: task-completion rate, regression catches,
   tool-call accuracy, and **router tier-selection accuracy**. Evals are the contract;
   no routing change ships without moving the numbers.
3. **Operational telemetry → analysis** — every task run is instrumented (tokens,
   latency, tier, QA outcome, retries); the data is analyzed to find which task features
   predict QA failure, and to close the loop back into the router.

The lifecycle/state machine (`file → start → QA → wrap`, git-worktree-per-task
isolation, pluggable QA tiers) is the substrate these three sit on.

## Status — honest, shipped vs. planned

This is **incubating**. What's real today vs. planned is tracked transparently in
[`TOOLING_ROADMAP.md`](./TOOLING_ROADMAP.md). Nothing below is claimed as working until
it has tests + evals behind it.

| Component | State |
|---|---|
| Task lifecycle workflow (incubated in a private app repo) | proven in daily use; being hardened + extracted here |
| Plugin package skeleton | filed (`future_devs/PLUGIN_PACKAGE_SKELETON_SPEC.md`) |
| `RightSizer` learned router | **planned** — design in `future_devs/AI_ENGINEERING_LAYER_SPEC.md` |
| Eval harness | **planned** — same spec |
| Telemetry + analysis | **planned** — same spec |

> Design-for-extraction invariant: no project-specific literal lives in the generic
> layer — build/test commands, QA tiers, and paths are per-project config.

## How it's meant to work

```
                 ┌──────────────────────────────────────────────┐
   a task  ─────▶│  RightSizer (learned)                         │
 (diff size,     │   features → model tier + effort + QA tier    │
  files, risk,   └───────────────┬──────────────────────────────┘
  domain)                        │ runs in an isolated git worktree
                                 ▼
                 ┌──────────────────────────────────────────────┐
                 │  lifecycle:  file → start → QA → wrap          │
                 │  pluggable QA tiers (lint → test → review …)  │
                 └───────────────┬──────────────────────────────┘
                                 │ QA pass/fail + cost/latency
                                 ▼
                 ┌──────────────────────────────────────────────┐
                 │  telemetry  →  eval harness  →  router update │  ◀── learning loop
                 └──────────────────────────────────────────────┘
```

## Where this fits in the landscape

The model-router idea exists in primitive/static form (Aider's architect/editor split;
Claude Code's per-agent `model`/`effort`), and fixed QA gates exist in spec-driven tools
(Spec-Kit, BMAD). The open ground this targets is the **adaptive, outcome-feedback-driven
combination across multiple projects** — and, just as importantly, doing it
**measurably** (evals + cost curves) rather than as static config.

## Distribution

Shipped as a **Claude Code plugin** via a git marketplace. Install (once a release exists):

```
/plugin marketplace add Omer-Hedvat/claude-task-lifecycle
/plugin install claude-task-lifecycle
```

## Repo map

| Path | What |
|---|---|
| `TASK_MGMT_HARDENING_SPEC.md` | Epic — harden the internal architecture (prerequisite) |
| `TASK_LIFECYCLE_PLUGIN_SPEC.md` | Epic — publishable multi-project plugin |
| `future_devs/AI_ENGINEERING_LAYER_SPEC.md` | Epic — the router + evals + telemetry (the AI-engineering core) |
| `TOOLING_ROADMAP.md` | Tracker for all epics + children |
| `docs/DESIGN.md` | Design notes / architecture decisions |
| `docs/eval-harness-design.md` | Build-ready design for the eval harness (the AI-Eng core) |
| `CLAUDE.md` | Working doc for agents operating in this repo |

## License

MIT — see [`LICENSE`](./LICENSE).
