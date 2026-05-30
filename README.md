# claude-task-lifecycle

A standalone, multi-project, adaptive task-management workflow for Claude Code —
extracted from the PowerME `.claude/` workflow (worktree-per-task isolation, the
`file → start → QA → wrap` lifecycle, 4-tier QA triage, model right-sizing, and
merge-first dedup at filing time).

The goal is a reusable **Claude Code plugin**, served from a private git marketplace,
that three separate repos consume — each declaring its own build/test commands, QA
tiers, and paths via per-project config while sharing the same git-based lifecycle
state machine.

## Status

Incubating. Two epics drive the work — see [`TOOLING_ROADMAP.md`](./TOOLING_ROADMAP.md):

1. **Task Management Architecture Hardening** (`TASK_MGMT_HARDENING_SPEC.md`) — harden
   the internal architecture (status off the merge path, one status enum, DRY plumbing,
   config-driven env, drift detection, state verbs). Prerequisite for extraction.
2. **Publishable `claude-task-lifecycle` Plugin** (`TASK_LIFECYCLE_PLUGIN_SPEC.md`) —
   carve the generic engine into a multi-project plugin and pilot it on two non-Android
   stacks. Depends on several hardening children.

> Incubation note: the hardening work currently lives inside the PowerME repo per the
> "file everything into PowerME for now, take it outside later" decision. This repo is
> the eventual standalone home; design every child for that extraction from day one —
> no PowerME literal in the generic layer.

## Target consumers

| Project | Stack | Build/test | QA tiers (project-declared) |
|---|---|---|---|
| PowerME | Android / Kotlin | Gradle + JUnit | Maestro + Gemini visual + hardware |
| personal | Python / data science | pytest / notebook runs | data validation, notebook-output diff |
| career-watch | website | npm build / vitest | Playwright / Lighthouse / visual |
