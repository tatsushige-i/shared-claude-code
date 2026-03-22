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
.github/
  ISSUE_TEMPLATE/   # 5 issue templates (Japanese)
  workflows/ci.yml  # CI: eol-check → lint → test → build
docs/ja-JP/     # Japanese translations (supplementary, not authoritative)
.claude/
  rules/        # Symlinks → ../../rules/
  skills/       # Symlinks → ../../skills/
```

## Adding a New Skill

1. Create `skills/<category>-<object>-<verb>/SKILL.md` with YAML frontmatter (`name`, `description`)
2. Add a symlink: `.claude/skills/<name> -> ../../skills/<name>`
3. Update `skills/README.md` with a row in the skills table
4. Add Japanese translation at `docs/ja-JP/skills/<name>/SKILL.md`
