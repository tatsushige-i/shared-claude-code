# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A centralized source of shared Claude Code rules and skills distributed to consuming projects. All consuming repositories must live in the same parent directory as this one.

**Distribution methods:**

- **Rules, skills, and agents** — distributed via symlinks into `.claude/rules/`, `.claude/skills/`, and `.claude/agents/` of consuming repos
- **GitHub configuration and CI templates** — distributed via file copy using the `/config-github-sync` skill

## Architecture

- **rules/** — Master rule files (English), symlinked into consuming repos
- **skills/** — Master skill definitions, symlinked into consuming repos
- **agents/** — Master subagent definitions (single `.md` files), symlinked into consuming repos
- **hooks/** — Shared hook definitions (`shared-hooks.json`), merged via `config-claude-sync`
- **.claude/** — Symlinks to `rules/`, `skills/`, and `agents/`, plus `settings.json` for this repo
- **ci-templates/** — CI/config templates by language, copied via `/config-github-sync`
- **.github/** — Issue/PR templates, copied via `/config-github-sync`
- **docs/ja-JP/** — Japanese translations (supplementary, not authoritative)

## Documentation Language (this repo)

All master `.md` files in this repository (rules, skills, agents, README, CLAUDE.md) must be written in **English**. Japanese translations are placed under `docs/ja-JP/` as supplementary documentation, and `docs/ja-JP/` is reserved for translations of English masters only.

## Adding a New Skill

When adding a skill, confirm with the user whether it is a **shared skill** or a **repo-local skill**.

### Shared Skill (distributed to consuming repos via symlinks)

1. Create `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`)
2. Add a symlink: `.claude/skills/<name> -> ../../skills/<name>`
3. Update `skills/README.md` with a row in the skills table
4. Add Japanese translation at `docs/ja-JP/skills/<name>/SKILL.md`

### Repo-Local Skill (used only in this repository)

Prefix the skill name with `local-` (e.g., `local-docs-validate`).

1. Create `.claude/skills/local-<name>/SKILL.md` directly with YAML frontmatter (`name`, `description`)
2. Add Japanese translation at `docs/ja-JP/local-skills/local-<name>/SKILL.md`

## Adding a New Rule

### Shared Rule (distributed to consuming repos via symlinks)

1. Create `rules/<name>.md`
2. Add a symlink: `.claude/rules/<name>.md -> ../../rules/<name>.md`
3. Add Japanese translation at `docs/ja-JP/rules/<name>.md`

## Adding a New Agent

A subagent is a single Markdown file with YAML frontmatter (`name`, `description`, and optionally `tools`, `model`) followed by the system prompt. The invocable identifier comes from the `name` field (not the filename) and must be unique within the scope. The `model` field may pin a model (`opus`, `sonnet`, `haiku`, `fable`, or `inherit`).

When adding an agent, confirm with the user whether it is a **shared agent** or a **repo-local agent**.

### Shared Agent (distributed to consuming repos via symlinks)

1. Create `agents/<name>.md` with YAML frontmatter (`name`, `description`)
2. Add a symlink: `.claude/agents/<name>.md -> ../../agents/<name>.md`
3. Update `agents/README.md` with a row in the agents table
4. Add Japanese translation at `docs/ja-JP/agents/<name>.md`

In consuming repos, shared agents are synced under `.claude/agents/shared/` (Claude Code scans `.claude/agents/` recursively, so the subdirectory does not affect invocation).

### Repo-Local Agent (used only in this repository)

Prefix the agent name with `local-` (e.g., `local-doc-reviewer`).

1. Create `.claude/agents/local-<name>.md` directly with YAML frontmatter (`name`, `description`)
2. Add Japanese translation at `docs/ja-JP/local-agents/local-<name>.md`

## Adding a New Hook

### Shared Hook (distributed to consuming repos via `config-claude-sync`)

Criteria: project-agnostic and reusable across any project.

1. Add an entry to `hooks/shared-hooks.json` (mirrors `settings.json` `hooks` structure; include `_id`, `_description`, `_detect_by` metadata fields)
2. Add the same hook to `.claude/settings.json` in this repository

### Repo-Local Hook (used only in this repository)

Criteria: specific to this repository's file structure, tools, or workflows.

1. Add only to `.claude/settings.json`
