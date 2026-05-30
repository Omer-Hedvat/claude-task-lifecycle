# Tooling Roadmap — claude-task-lifecycle

Tracker for the two Tooling epics that build this project. Status enum:
`not-started` · `in-progress` · `completed` (awaiting QA) · `wrapped` · `blocked`.

---

## 🏔️ Epics

| Status | Phase | Name | Spec | Remaining work | Rollup |
|---|---|---|---|---|---|
| `not-started` | Tooling | Task Management Architecture Hardening | `TASK_MGMT_HARDENING_SPEC.md` | 8 children, none started — start with `status_off_merge_path` (A1) | 0/8 wrapped · 8 not-started |
| `not-started` | Tooling | Publishable `claude-task-lifecycle` Plugin | `TASK_LIFECYCLE_PLUGIN_SPEC.md` | 7 children, blocked on hardening children landing first (skeleton scaffold can start now) | 0/7 wrapped · 7 not-started |
| `not-started` | AI Engineering | AI Engineering Layer — learned router + evals + telemetry | `future_devs/AI_ENGINEERING_LAYER_SPEC.md` | 3 children — the portfolio centerpiece; needs telemetry hook points from hardening | 0/3 wrapped · 3 not-started |

---

## 1. Task Management Architecture Hardening — children

| # | Status | Slug | What |
|---|---|---|---|
| A1 | `not-started` | `status_off_merge_path` | Move mutable status out of branch-merged markdown (single `main`-only source / harness tasks). Root fix — do first. |
| — | `not-started` | `unify_status_vocabulary` | One status enum shared by bugs + features; presentation layer maps to emoji. (After A1.) |
| — | `not-started` | `dry_skill_plumbing` | Extract repeated git/build steps into `GIT_PROTOCOL.md` / `BUILD_PROTOCOL.md`; skills reference, not restate. (Parallel-safe.) |
| — | `not-started` | `parameterize_environment` | Remove machine-specific literals (gradle path, `JAVA_HOME`, username, dev UID) → config. (Parallel-safe.) |
| — | `not-started` | `fix_coauthor_trailer` | Make `Co-Authored-By` trailer a config value, not hardcoded model name. (After parameterize.) |
| — | `not-started` | `tasks_doctor_skill` | Drift/orphan detection across tracker status, branches, worktrees, specs. (Parallel-safe.) |
| — | `not-started` | `task_state_verbs` | First-class `block` / `unblock` / `abandon` verbs over the unified enum. (After A1 + unify.) |
| — | `not-started` | `move_recommendation_protocol` | Move `tasks_status` recommendation protocol from `CLAUDE.md` into the skill it governs. (Parallel-safe.) |

**Phasing:** A1 first → parallel early wins (`dry_skill_plumbing`, `parameterize_environment`, `tasks_doctor_skill`, `move_recommendation_protocol`) → `unify_status_vocabulary` → `task_state_verbs`; `fix_coauthor_trailer` after parameterize.

---

## 2. Publishable `claude-task-lifecycle` Plugin — children

Depends on the hardening epic. Phasing order below.

| # | Status | Slug | What |
|---|---|---|---|
| 0 | `not-started` | `plugin_package_skeleton` | Scaffold empty-but-valid plugin package structure (`.claude-plugin/`, `skills/`, `hooks/`, `agents/`). Precedes extraction. Spec: `future_devs/PLUGIN_PACKAGE_SKELETON_SPEC.md`. |
| 1 | `not-started` | (hardening epic) | Core engine done right — status model, config-driven env, doctor, verbs. |
| 2 | `not-started` | `plugin_config_schema` | Define per-project `userConfig` contract in `plugin.json`. (Parallel-safe with QA tiers.) |
| 2 | `not-started` | `plugin_pluggable_qa_tiers` | Tier 1 generic; Tiers 2–4 project-declared. (Parallel-safe with config schema.) |
| 3 | `not-started` | `plugin_state_backend_adapter` | Task state behind an interface (markdown + JSON adapters). Needs `status_off_merge_path`. |
| 4 | `not-started` | `plugin_extract_skeleton` | Carve generic skills out. Needs config + QA tiers + DRY plumbing. |
| 5 | `not-started` | `plugin_pilot_adoption` | Wire `career-watch` + `personal` via the private marketplace; prove adaptivity. |
| 6 | `not-started` | `plugin_packaging_docs` | Package + document; standalone-repo extraction + PowerME migration (post-launch). |

---

## 3. AI Engineering Layer — children

The portfolio centerpiece. Spec: `future_devs/AI_ENGINEERING_LAYER_SPEC.md`.

| # | Status | Slug | What |
|---|---|---|---|
| 1 | `not-started` | `rightsizer_router` | Learned model-tier/effort/QA router; rules baseline (v0) → learned (v1). Deliverable: cost-vs-quality curve. **Design ready:** `docs/rightsizer-router-design.md` (phases R0–R4). |
| 2 | `not-started` | `eval_harness` | Golden set + deterministic checks + LLM-as-judge; metrics + CI gate. Deliverable: before/after table. **Design ready:** `docs/eval-harness-design.md` (phases E0–E4). |
| 3 | `not-started` | `telemetry_analysis` | Instrument runs (opt-in); analyze which features predict QA failure. Deliverable: analysis writeup; feeds router v1. |
