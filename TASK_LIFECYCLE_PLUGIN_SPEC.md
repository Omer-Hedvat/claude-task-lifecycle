# Publishable `claude-task-lifecycle` Plugin (multi-project, adaptive)

| Field | Value |
|---|---|
| **Type** | Epic |
| **Phase** | Tooling |
| **Status** | `not-started` |
| **Tracker** | `TOOLING_ROADMAP.md` |
| **Depends on** | `TASK_MGMT_HARDENING_SPEC.md` (several children) |
| **Children** | 6 tasks |
| **Rollup** | 0/6 wrapped · 6 not-started |

> **Incubation note:** This is filed and incubated inside the PowerME repo for now (per the developer's decision to "file everything into PowerME for now, take it outside later"). The end state is a **standalone repo** consumed by all projects; design every child for that extraction from day one — no PowerME literal may be hardwired into the generic layer.

---

## Overview

The PowerME task-management workflow has clearly-transferable, best-in-class pieces: worktree-per-task isolation, the 4-tier QA triage ("lowest tier that can verify"), the model right-sizing table, merge-first dedup at filing time, and the `file → start → QA → wrap` lifecycle. These generalize well beyond Android/PowerME.

The goal of this Epic is to turn the workflow into a **standalone, multi-project, adaptive tool** that three separate repos consume:

| Project | Stack | Build/test | QA tiers (project-declared) |
|---|---|---|---|
| PowerME | Android / Kotlin | Gradle + JUnit | Maestro + Gemini visual + hardware |
| personal | Python / data science | pytest / notebook runs | data validation, notebook-output diff |
| career-watch | website | npm build / vitest | Playwright / Lighthouse / visual |

The **constant** across all three is git: worktree isolation, the lifecycle state machine, status transitions, model right-sizing, and the QA *triage philosophy*. The **variable** is everything else (build/test commands, artifact type, the concrete QA tiers, paths) — all of which moves into per-project configuration and project-declared tier definitions.

## Locked decisions

- **Distribution = Claude Code plugin, served from a private git marketplace.** Verified the only mechanism that bundles hooks + supports per-project config + version pinning + an update path. (Sources: code.claude.com/docs `plugins`, `plugin-marketplaces`, `plugins-reference`.)
- **Per-project config = `userConfig` in `plugin.json`** (prompts on enable; values exposed as `${user_config.*}` in skill/hook/MCP content and `CLAUDE_PLUGIN_OPTION_*` env in hook subprocesses). For richer/structured config, bundled hook scripts read `${CLAUDE_PROJECT_DIR}/.claude/<tool>-config.json` from the consuming repo.
- **Task state = per-project local** — each repo owns its own tracker (git-co-located). No central store to start; an optional read-only cross-project dashboard is a later add-on.
- **Adoption order = low-stakes first** — adopt on `career-watch` + `personal` first; migrate PowerME only **after the AAB ships**. Never destabilize the launch sprint with a workflow swap.
- **Guardrail hooks ship in the plugin** — e.g. a `PreToolUse` hook blocking direct commit/push to the default branch (the exact guard that surfaced during filing).

## Architecture

- **Plugin package:** `skills/` (the lifecycle commands), `hooks/hooks.json` (guardrails), `agents/` (git-worker + plan agents), optional `.mcp.json` / `monitors/`. A `.claude-plugin/plugin.json` declares `userConfig` and `version`.
- **Private marketplace:** a git repo with `.claude-plugin/marketplace.json` listing the plugin; each project registers it via `extraKnownMarketplaces` + `enabledPlugins` in `.claude/settings.json`; version pinned via `ref`/`sha`.
- **Pluggable QA tiers:** Tier 1 generic ("run configured test/build + greps"); Tiers 2–4 project-declared.
- **State backend interface:** local markdown + JSON adapters first; generic skills call the interface, never a concrete store.

---

## Phasing

1. Core engine done right (the `TASK_MGMT_HARDENING_SPEC.md` children) — status model, config-driven env, doctor, verbs.
2. `plugin_config_schema` + `plugin_pluggable_qa_tiers` — define the contracts (parallel-safe).
3. `plugin_state_backend_adapter` — state behind an interface (needs `status_off_merge_path`).
4. `plugin_extract_skeleton` — carve the generic skills out (needs config + QA tiers + DRY plumbing).
5. `plugin_pilot_adoption` — wire `career-watch` + `personal` to the plugin via the private marketplace; prove adaptivity on two non-Android stacks.
6. `plugin_packaging_docs` — package + document; the standalone-repo extraction + PowerME migration happen here (post-launch).

---

## Invariants (every child must honor)

- The extracted plugin must reproduce the PowerME lifecycle exactly when given PowerME's config — no behavior regression for the home project.
- No project-specific literal may survive in the generic skeleton; all such values live in `userConfig` / project config / project-declared tiers.
- Every contract (config schema, QA tier interface, state backend interface) ships with at least one worked implementation per pilot stack as a conformance example.
- Design for extraction from day one — even while incubated in the PowerME repo, the generic layer must stay repo-agnostic.
