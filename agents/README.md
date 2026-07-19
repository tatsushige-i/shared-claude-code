# Agents

Shared subagent (agent) definitions distributed to consuming repositories via symlinks. Each agent is a single Markdown file (`agents/<name>.md`) with YAML frontmatter (`name`, `description`, and optionally `tools`, `model`) followed by the system prompt. Agents are invoked as a `subagent_type` through the Agent tool — including from within skills. Use `/config-claude-sync` to detect missing symlinks and sync them automatically.

In consuming repositories, shared agents are symlinked under `.claude/agents/shared/`. Claude Code scans `.claude/agents/` recursively, so the `shared/` subdirectory does not affect how an agent is identified or invoked — identity comes solely from the `name` frontmatter field, which must be unique within the scope.

| Agent | Description |
|---|---|
| `review-correctness` | Code reviewer focused on correctness and bugs (opus). Part of the `review-team-run` parallel team. |
| `review-security` | Code reviewer focused on security (opus). Part of the `review-team-run` parallel team. |
| `review-design` | Code reviewer focused on design and simplicity (opus). Part of the `review-team-run` parallel team. |
| `review-readability` | Code reviewer focused on readability and cleanliness (sonnet). Part of the `review-team-run` parallel team. |
| `review-tests` | Code reviewer focused on testing (sonnet). Part of the `review-team-run` parallel team. |
| `impl-planner` | Read-only implementation planner (opus) — lays out affected files, steps, and decision points from an Issue. Used by the `impl-pipeline` skill. |
| `impl-coder` | Autonomous implementer (sonnet) — executes a confirmed plan for mechanical Issues. Used by the `impl-pipeline` skill. |
| `impl-tester` | Test author (sonnet) — writes and runs tests from the spec, not the implementation. Used by the `impl-pipeline` skill. |
