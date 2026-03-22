# Skills

Shared skills distributed to consuming repositories via symlinks. Each skill is invoked as a slash command in Claude Code. Use `/config-claude-sync` to detect missing symlinks and sync them automatically.

| Skill | Command | Description |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | Detect missing symlinks and sync rules/skills under `.claude/` |
| `config-github-sync` | `/config-github-sync` | Detect diffs and copy-sync ISSUE_TEMPLATE/workflows under `.github/` |
| `git-branch-cleanup` | `/git-branch-cleanup` | Local branch cleanup (switch to main, delete branches, pull) |
| `git-issue-create` | `/git-issue-create` | Create Issue from conversation context (title, body, label inference, preview) |
| `git-issue-start` | `/git-issue-start <Issue#>` | Fetch Issue, validate labels, create branch, enter Plan Mode |
| `git-pr-create` | `/git-pr-create` | Identify Issue, check size limits, analyze diff, create PR |
| `git-review-respond` | `/git-review-respond <PR#>` | Analyze review comments, fix code, reply |
| `tech-debt-audit-nextjs` | `/tech-debt-audit-nextjs` | Audit technical debt in Next.js (App Router) projects with prioritized report |
