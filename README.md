**Language:** English | [日本語](./docs/ja-JP/README.md)

# shared-claude-code

A centralized source of shared Claude Code rules, skills, and agents for use across multiple repositories. Rules, skills, and agents are distributed via **symlinks**, while GitHub configuration and CI templates are distributed via **file copy**.

## Structure

```text
rules/                  # Master rule files (English) — symlinked into consuming repos
├── conventions.md      # Development conventions
├── security.md         # Security rules
└── tech-debt-checklist.md # Technical debt checklist
skills/                 # Master skill definitions — symlinked into consuming repos
├── README.md           # Skills index table
├── config-claude-sync/ # Sync shared rules/skills via symlinks
├── config-github-sync/ # Sync .github files and CI templates via file copy
├── git-branch-cleanup/ # Worktree-scoped cleanup after PR merge
├── git-issue-create/   # Create GitHub Issue from conversation context
├── git-issue-start/    # Start workflow from GitHub Issue
├── git-pr-create/      # Create GitHub PR with analysis
├── git-pr-finalize/    # Watch CI/review, address findings, merge, clean up
├── git-review-respond/ # Respond to PR review comments
├── tech-debt-audit-flutter/ # Audit technical debt in Flutter / Dart projects
└── tech-debt-audit-nextjs/ # Audit technical debt in Next.js projects
agents/                 # Master subagent definitions (single .md files) — symlinked into consuming repos
└── README.md           # Agents index table
hooks/                  # Shared hook definitions — merged into consuming repos' settings.json via config-claude-sync
└── shared-hooks.json   # Shared hooks for PreToolUse / Stop / UserPromptSubmit / etc.
ci-templates/           # CI/config templates by language — copied to consuming repos
├── flutter/            # Flutter template (test.yml: format / analyze / test / build_runner)
└── nextjs/             # Next.js template (ESLint, Jest, TypeScript configs)
.github/
└── ISSUE_TEMPLATE/     # Issue templates (Japanese) — copied to consuming repos
docs/ja-JP/             # Japanese translations (supplementary, not authoritative)
.claude/
├── rules/              # Symlinks → ../../rules/
├── skills/             # Symlinks → ../../skills/ + repo-local skills
└── agents/             # Symlinks → ../../agents/ + repo-local agents
```

## Getting Started

**Prerequisite**: All consuming repositories must be in the same parent directory as this repository.

### 1. Symlink rules, skills, and agents

From the consuming repository root, create symlinks to shared rules, skills, and agents:

```bash
# Create destination directories
mkdir -p .claude/rules/shared .claude/skills .claude/agents/shared

# Rules
ln -s ../../../../shared-claude-code/rules/conventions.md .claude/rules/shared/conventions.md
ln -s ../../../../shared-claude-code/rules/security.md .claude/rules/shared/security.md
ln -s ../../../../shared-claude-code/rules/tech-debt-checklist.md .claude/rules/shared/tech-debt-checklist.md

# Skills
ln -s ../../../shared-claude-code/skills/config-claude-sync .claude/skills/config-claude-sync
ln -s ../../../shared-claude-code/skills/config-github-sync .claude/skills/config-github-sync
ln -s ../../../shared-claude-code/skills/git-branch-cleanup .claude/skills/git-branch-cleanup
ln -s ../../../shared-claude-code/skills/git-issue-create .claude/skills/git-issue-create
ln -s ../../../shared-claude-code/skills/git-issue-start .claude/skills/git-issue-start
ln -s ../../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
ln -s ../../../shared-claude-code/skills/git-pr-finalize .claude/skills/git-pr-finalize
ln -s ../../../shared-claude-code/skills/git-review-respond .claude/skills/git-review-respond
ln -s ../../../shared-claude-code/skills/tech-debt-audit-flutter .claude/skills/tech-debt-audit-flutter
ln -s ../../../shared-claude-code/skills/tech-debt-audit-nextjs .claude/skills/tech-debt-audit-nextjs

# Agents (symlinked under .claude/agents/shared/; add a line per shared agent as they are introduced)
# ln -s ../../../../shared-claude-code/agents/<name>.md .claude/agents/shared/<name>.md
```

Or use the `/config-claude-sync` skill to detect missing symlinks and create them automatically.

The `/config-claude-sync` skill also merges shared hook entries from `hooks/shared-hooks.json` into the consuming repository's `.claude/settings.json` (existing entries are preserved; only new or updated hooks are applied). No manual symlinking is required for hooks.

### 2. Copy GitHub configuration and CI templates

Use the `/config-github-sync` skill to copy Issue templates, workflow files, and CI configuration templates to your repository.

The shared workflows include `close-linked-issues-on-develop.yml`, which automatically closes the linked Issues (`Closes #N`, etc.) when a PR targeting `develop` is merged — covering the case GitHub's native auto-close does not handle for non-default branches. It does nothing in repositories that do not use a `develop` branch.

## Available Skills

| Skill | Command | Description |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | Detect missing symlinks and sync rules/skills under `.claude/`, and merge shared hooks into `settings.json` |
| `config-github-sync` | `/config-github-sync` | Detect diffs and copy-sync ISSUE_TEMPLATE/workflows under `.github/` |
| `git-branch-cleanup` | `/git-branch-cleanup` | Worktree-scoped cleanup — remove current worktree and its working branch |
| `git-issue-create` | `/git-issue-create` | Create Issue from conversation context (title, body, label inference, preview) |
| `git-issue-start` | `/git-issue-start <Issue#>` | Fetch Issue, validate labels, create branch, enter Plan Mode |
| `git-pr-create` | `/git-pr-create` | Identify Issue, check size limits, analyze diff, create PR |
| `git-pr-finalize` | `/git-pr-finalize [PR#]` | Watch CI/Copilot review, address findings, merge after confirmation, clean up branches |
| `git-review-respond` | `/git-review-respond <PR#>` | Analyze review comments, fix code, reply |
| `tech-debt-audit-flutter` | `/tech-debt-audit-flutter` | Audit technical debt in Flutter / Dart projects with prioritized report |
| `tech-debt-audit-nextjs` | `/tech-debt-audit-nextjs` | Audit technical debt in Next.js (App Router) projects with prioritized report |
