# Agents

Shared subagent (agent) definitions distributed to consuming repositories via symlinks. Each agent is a single Markdown file (`agents/<name>.md`) with YAML frontmatter (`name`, `description`, and optionally `tools`, `model`) followed by the system prompt. Agents are invoked as a `subagent_type` through the Agent tool — including from within skills. Use `/config-claude-sync` to detect missing symlinks and sync them automatically.

In consuming repositories, shared agents are symlinked under `.claude/agents/shared/`. Claude Code scans `.claude/agents/` recursively, so the `shared/` subdirectory does not affect how an agent is identified or invoked — identity comes solely from the `name` frontmatter field, which must be unique within the scope.

| Agent | Description |
|---|---|
| _(none yet)_ | Shared agents will be listed here as they are added. |
