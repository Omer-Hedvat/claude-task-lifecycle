# Design notes

Short architecture-decision record for `claude-task-lifecycle`. Written to be read by a
human inspecting the repo, and by an agent operating in it.

## Problem framing

Driving AI coding agents across multiple real repos is an **orchestration + cost +
quality** problem, not a prompting problem. Three decisions repeat on every task:

1. **Which model tier / effort** is enough? (cost vs. quality)
2. **How much QA** is warranted? (a typo fix and a schema migration should not get the
   same gate)
3. **In what isolation** does it run, so parallel work doesn't collide?

Most existing tools fix these as static config. The thesis here is that (1) and (2)
should be **adaptive and measured**, learned from outcomes.

## Key decisions

| # | Decision | Why |
|---|---|---|
| D1 | Git is the substrate: worktree-per-task isolation, lifecycle co-located with code | Reuses git's branching/reflog safety; no external state store to start |
| D2 | Task status leaves the branch-merge path (single writer) | Avoids mirror-commit churn and forced merge-conflict resolution rituals |
| D3 | One status enum; presentation maps to emoji | Two vocabularies (bugs vs. features) was accidental complexity |
| D4 | Model right-sizing is a **router with a rules baseline first**, learned later | The baseline is the control; the learned model's lift must be measurable, not assumed |
| D5 | QA is **tiered and pluggable** (Tier 1 generic; 2–4 project-declared) | "Lowest tier that can verify" — match QA cost to task risk |
| D6 | Everything project-specific is config / project-declared | Design-for-extraction: the generic layer must stay repo-agnostic |
| D7 | No metric without an eval/telemetry behind it | Honesty over hype; this is the differentiator vs. capability-claim README projects |

## Layering

```
project config (build/test cmds, QA tiers, paths)        ← variable, per repo
        │
RightSizer router  ──uses──▶  lifecycle state machine    ← constant across repos
        ▲                              │
        └──────── telemetry ◀──────────┘  (features → QA outcome → router training)
                      │
                  eval harness  (golden set + LLM-as-judge → metrics → CI gate)
```

## Open questions

- Router v1 model class: small classifier vs. an LLM-as-router prompt — decide by measuring
  cost/latency/accuracy of each against the v0 rules baseline.
- State backend: markdown vs. JSON vs. harness task tools — behind an interface so it can
  change without touching the generic skills (D6).
- How much telemetry is enough to train a useful router without over-collecting.

See `TOOLING_ROADMAP.md` for the sequenced work and `future_devs/AI_ENGINEERING_LAYER_SPEC.md`
for the AI-engineering core.
