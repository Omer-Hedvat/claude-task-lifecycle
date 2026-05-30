# Task Management Architecture Hardening

| Field | Value |
|---|---|
| **Type** | Epic |
| **Phase** | Tooling |
| **Status** | `not-started` |
| **Tracker** | `TOOLING_ROADMAP.md` |
| **Children** | 8 tasks |
| **Rollup** | 0/8 wrapped · 8 not-started |

---

## Overview

The PowerME task-management workflow (the `.claude/` skills + `BUG_TRACKER.md` / `ROADMAP.md` trackers + git-worktree-per-task model) is sophisticated and effective, but a code review surfaced structural fragilities that grow more expensive as the workflow scales. This Epic hardens the *internal* architecture — it does not change the developer-facing surface beyond what each child requires. Publishing the workflow as a reusable plugin is a separate initiative (`TASK_LIFECYCLE_PLUGIN_SPEC.md`), which depends on several of these children landing first.

The findings this Epic addresses, from highest to lowest leverage:

1. **Task status lives inside the git merge path.** Status is stored in `BUG_TRACKER.md` / `ROADMAP.md` tables that are edited on every branch, mirrored to `main`, and then resolved through an *expected* 3-way merge conflict via a blanket "always take the task branch version" rule. This risks silently clobbering legitimate concurrent edits and litters `main` with mirror commits.
2. **Two status vocabularies** (`🔵 Open`/`🟡 In Progress`/… for bugs vs. `not-started`/`in-progress`/… for features) that mean the same thing.
3. **Repeated git/build plumbing** restated across `start_epic`, `start_batch`, `start_qa`, `wrap_task` → drift risk.
4. **Hardcoded environment** (absolute gradle path, `JAVA_HOME`, username, dev UID) baked into skill bodies.
5. **Hardcoded `Co-Authored-By: Sonnet 4.6` trailer** even when sessions run on other models.
6. **No drift/orphan detection** — tracker status, git branches, worktrees, and spec files drift apart (e.g. orphaned `PowerME-file-*` worktrees, tasks marked In Progress with no branch).
7. **No first-class `block`/`unblock`/`abandon` verbs** — those transitions are manual table edits.
8. **The `tasks_status` recommendation protocol lives in `CLAUDE.md`** instead of in the skill it governs.

---

## Architecture

- **Single source of truth for volatile status.** Move the mutable status field out of branch-merged markdown into a surface that only `main` writes (a single status file on `main`, or the harness `TaskCreate`/`TaskUpdate` tools). Spec text and bug descriptions can remain in-branch; only the status field must leave the merge path. This eliminates both the mirror commits and the conflict-resolution ritual.
- **One status enum** shared by bugs and features, with a presentation layer mapping to emojis if desired.
- **Shared protocol docs** (`GIT_PROTOCOL.md`, `BUILD_PROTOCOL.md`) that every skill references rather than restates.
- **Config-driven environment** so no machine-specific literal appears in a skill body (prerequisite for the plugin Epic).

---

## Phasing

- **First:** `status_off_merge_path` (A1) — the root fragility; several other items simplify once status leaves the merge path.
- **Parallel-safe early wins:** `dry_skill_plumbing`, `parameterize_environment`, `tasks_doctor_skill`, `move_recommendation_protocol` (disjoint surfaces).
- **After A1:** `unify_status_vocabulary` → then `task_state_verbs` (verbs operate on the unified enum).
- **After parameterize:** `fix_coauthor_trailer` (trailer becomes a config value).

---

## Invariants (every child must honor)

- No child may regress the existing lifecycle: `file → start → QA → wrap` must keep working end-to-end after each child lands.
- No machine-specific literal introduced or left in a skill body once `parameterize_environment` lands.
- Every skill that mutates status must go through the single source of truth defined by `status_off_merge_path` once it lands.
- Changes are documentation/skill-only (`.claude/`, root tracker/spec files) — no `app/` source changes.
- **Design for extraction.** Each fix here becomes a feature of the standalone multi-project tool (`TASK_LIFECYCLE_PLUGIN_SPEC.md`). Build it repo-agnostic — prefer config-driven values over PowerME literals even while incubating in this repo, so the eventual extraction is mechanical.
