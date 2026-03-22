**Language:** English | [日本語](./docs/ja-JP/README.md)

# shared-claude-code

A centralized source of shared Claude Code rules and skills for use across multiple repositories. Rules and skills are distributed via **symlinks**, while GitHub configuration and CI templates are distributed via **file copy**.

## Structure

```
rules/                  # Master rule files (English) — symlinked into consuming repos
├── conventions.md      # Development conventions
└── security.md         # Security rules
skills/                 # Master skill definitions — symlinked into consuming repos
├── README.md           # Skills index table
├── config-claude-sync/ # Sync shared rules/skills via symlinks
├── config-github-sync/ # Sync .github files and CI templates via file copy
├── git-branch-cleanup/ # Local branch cleanup after PR merge
├── git-issue-create/   # Create GitHub Issue from conversation context
├── git-issue-start/    # Start workflow from GitHub Issue
├── git-pr-create/      # Create GitHub PR with analysis
└── git-review-respond/ # Respond to PR review comments
ci-templates/           # CI/config templates by language — copied to consuming repos
└── nextjs/             # Next.js template (ESLint, Jest, TypeScript configs)
.github/
└── ISSUE_TEMPLATE/     # Issue templates (Japanese) — copied to consuming repos
docs/ja-JP/             # Japanese translations (supplementary, not authoritative)
.claude/
├── rules/              # Symlinks → ../../rules/
└── skills/             # Symlinks → ../../skills/ + repo-local skills
```

## Getting Started

**Prerequisite**: All consuming repositories must be in the same parent directory as this repository.

### 1. Symlink rules and skills

From the consuming repository root, create symlinks to shared rules and skills:

```bash
# Rules
ln -s ../../shared-claude-code/rules/conventions.md .claude/rules/shared/conventions.md
ln -s ../../shared-claude-code/rules/security.md .claude/rules/shared/security.md

# Skills
ln -s ../../shared-claude-code/skills/config-claude-sync .claude/skills/config-claude-sync
ln -s ../../shared-claude-code/skills/config-github-sync .claude/skills/config-github-sync
ln -s ../../shared-claude-code/skills/git-branch-cleanup .claude/skills/git-branch-cleanup
ln -s ../../shared-claude-code/skills/git-issue-create .claude/skills/git-issue-create
ln -s ../../shared-claude-code/skills/git-issue-start .claude/skills/git-issue-start
ln -s ../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
ln -s ../../shared-claude-code/skills/git-review-respond .claude/skills/git-review-respond
```

Or use the `/config-claude-sync` skill to detect missing symlinks and create them automatically.

### 2. Copy GitHub configuration and CI templates

Use the `/config-github-sync` skill to copy Issue templates, workflow files, and CI configuration templates to your repository.

## Available Skills

| Skill | Command | Description |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | Detect missing symlinks and sync rules/skills under `.claude/` |
| `config-github-sync` | `/config-github-sync` | Detect diffs and copy-sync ISSUE_TEMPLATE/workflows under `.github/` |
| `git-branch-cleanup` | `/git-branch-cleanup` | Local branch cleanup (switch to main, delete branches, pull) |
| `git-issue-create` | `/git-issue-create` | Create Issue from conversation context (title, body, label inference, preview) |
| `git-issue-start` | `/git-issue-start <Issue#>` | Fetch Issue, validate labels, create branch, enter Plan Mode |
| `git-pr-create` | `/git-pr-create` | Identify Issue, check size limits, analyze diff, create PR |
| `git-review-respond` | `/git-review-respond <PR#>` | Analyze review comments, fix code, reply |
