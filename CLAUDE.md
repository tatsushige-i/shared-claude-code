# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A centralized source of shared Claude Code rules and skills distributed to consuming projects via symlinks. All consuming repositories must live in the same parent directory as this one.

**Symlink pattern** (from a consuming repo):
```bash
ln -s ../../shared-claude-code/rules/conventions.md .claude/rules/conventions.md
ln -s ../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
```

## Repository Structure

```
rules/          # Master rule files (English) — symlinked into .claude/rules/ of consuming repos
skills/         # Master skill definitions — symlinked into .claude/skills/ of consuming repos
  README.md     # Skills index table — must be updated when adding a skill
ci-templates/   # CI/config templates by language — copy to consuming repos
  nextjs/       # Next.js template (ci.yml + package.json + config files)
.github/
  ISSUE_TEMPLATE/   # 5 issue templates (Japanese)
docs/ja-JP/     # Japanese translations (supplementary, not authoritative)
.claude/
  rules/        # Symlinks → ../../rules/
  skills/       # Symlinks → ../../skills/
```

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
