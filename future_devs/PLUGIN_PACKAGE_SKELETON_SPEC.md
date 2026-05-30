# Scaffold the plugin package skeleton

| Field | Value |
|---|---|
| **Type** | Feature (child) |
| **Phase** | Tooling |
| **Status** | `not-started` |
| **Tracker** | `TOOLING_ROADMAP.md` |
| **Parent Epic** | `TASK_LIFECYCLE_PLUGIN_SPEC.md` |
| **Slug** | `plugin_package_skeleton` |
| **Effort** | XS |
| **Depends on** | none (pure scaffolding; precedes `plugin_extract_skeleton`) |

---

## Overview

Create the empty-but-valid plugin package structure in this repo so later children
(`plugin_config_schema`, `plugin_pluggable_qa_tiers`, `plugin_extract_skeleton`,
`plugin_packaging_docs`) have a home to fill. This is structure only — no generic
skills are carved out of PowerME here (that is `plugin_extract_skeleton`).

## Scope (from `TASK_LIFECYCLE_PLUGIN_SPEC.md` → Architecture)

Create:

- `.claude-plugin/plugin.json` — declares `name`, `version`, and a stub `userConfig`
  block (the real schema is `plugin_config_schema`).
- `.claude-plugin/marketplace.json` — private-marketplace manifest listing this plugin.
- `skills/` — placeholder for the lifecycle commands (`.gitkeep`).
- `hooks/hooks.json` — stub with the planned default-branch guardrail `PreToolUse` hook
  entry (no script wired yet).
- `agents/` — placeholder for git-worker + plan agents (`.gitkeep`).
- Optional: `.mcp.json` / `monitors/` placeholders.

## Out of scope

- Any real skill / hook / agent logic (later children).
- Any `userConfig` schema design (that is `plugin_config_schema`).
- Wiring consuming repos to the marketplace (that is `plugin_pilot_adoption`).

## Done when

- The directory structure exists and is committed.
- `plugin.json` and `marketplace.json` are syntactically valid JSON.
- Nothing references a PowerME-specific literal (design-for-extraction invariant).
