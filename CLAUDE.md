# claude-task-lifecycle — Claude Working Document

## Current State

**Project:** `claude-task-lifecycle` — a standalone, multi-project, adaptive
task-management workflow for Claude Code, extracted from the PowerME `.claude/`
workflow. End state: a Claude Code **plugin** served from a private git marketplace,
consumed by multiple repos (PowerME, `personal`, `career-watch`).

**Stage:** Incubating — repo scaffolded, no plugin code carved out yet. The hardening
work is currently being incubated inside the PowerME repo and migrates here later.

**Constant across consumers:** git — worktree isolation, the lifecycle state machine
(`file → start → QA → wrap`), status transitions, model right-sizing, and the QA
triage philosophy ("lowest tier that can verify").

**Variable per consumer:** build/test commands, artifact type, concrete QA tiers,
paths — all moves into per-project `userConfig` and project-declared tier definitions.

## Tracking

- `TOOLING_ROADMAP.md` — the two epics and their children. Check it before starting work.
- `TASK_MGMT_HARDENING_SPEC.md` — Epic 1 (internal hardening; prerequisite).
- `TASK_LIFECYCLE_PLUGIN_SPEC.md` — Epic 2 (publishable plugin; depends on Epic 1).

## Invariants

- **Design for extraction from day one.** No project-specific literal may survive in the
  generic layer — all such values live in `userConfig` / project config / project-declared
  tiers.
- The extracted plugin must reproduce the PowerME lifecycle exactly when given PowerME's
  config — no behavior regression for the home project.
- Every contract (config schema, QA tier interface, state backend interface) ships with at
  least one worked implementation per pilot stack as a conformance example.
- Adopt low-stakes repos first (`career-watch` + `personal`); migrate PowerME only after
  its AAB ships.

## Locked decisions

- Distribution = Claude Code plugin from a private git marketplace.
- Per-project config = `userConfig` in `plugin.json` (+ optional `.claude/<tool>-config.json`
  read by bundled hooks for richer/structured config).
- Task state = per-project local (git-co-located); no central store to start.
- Guardrail hooks ship in the plugin (e.g. a `PreToolUse` hook blocking direct commit/push
  to the default branch).
