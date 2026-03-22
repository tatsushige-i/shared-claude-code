# shared-claude-code

Shared Claude Code rules and skills for use across multiple repositories via symlinks.

## Structure

```
rules/
├── conventions.md          # Development conventions
└── security.md             # Security rules
skills/
├── config-claude-sync/     # Sync shared rules/skills from shared-claude-code via symlinks
├── config-github-sync/     # Sync .github files from shared-claude-code via file copy
├── git-branch-cleanup/     # Local branch cleanup after PR merge
├── git-issue-create/       # Create GitHub Issue from conversation context
├── git-issue-start/        # Start workflow from GitHub Issue
├── git-pr-create/          # Create GitHub PR with analysis
└── git-review-respond/     # Respond to PR review comments
```

## Usage

Each consuming repository creates symlinks from `.claude/rules/` and `.claude/skills/` to this repository:

```bash
# From the consuming repository root
ln -s ../../shared-claude-code/rules/conventions.md .claude/rules/conventions.md
ln -s ../../shared-claude-code/rules/security.md .claude/rules/security.md
ln -s ../../shared-claude-code/skills/config-claude-sync .claude/skills/config-claude-sync
ln -s ../../shared-claude-code/skills/config-github-sync .claude/skills/config-github-sync
ln -s ../../shared-claude-code/skills/git-branch-cleanup .claude/skills/git-branch-cleanup
ln -s ../../shared-claude-code/skills/git-issue-create .claude/skills/git-issue-create
ln -s ../../shared-claude-code/skills/git-issue-start .claude/skills/git-issue-start
ln -s ../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
ln -s ../../shared-claude-code/skills/git-review-respond .claude/skills/git-review-respond
```

**Prerequisite**: All repositories must be in the same parent directory.
