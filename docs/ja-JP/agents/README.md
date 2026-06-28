# Agents

シンボリックリンクで消費リポジトリに配布する、共有 subagent（エージェント）定義。各エージェントは YAML フロントマター（`name`, `description`、任意で `tools`, `model`）とシステムプロンプト本文からなる単一の Markdown ファイル（`agents/<name>.md`）。エージェントは Agent ツールの `subagent_type` として（スキル内からも）起動できる。不足しているシンボリックリンクは `/config-claude-sync` で自動検出・同期できる。

消費リポジトリでは、共有エージェントは `.claude/agents/shared/` 配下にシンボリックリンクされる。Claude Code は `.claude/agents/` を再帰的に走査するため、`shared/` サブディレクトリはエージェントの識別・起動方法に影響しない。識別子はフロントマターの `name` フィールドのみで決まり、スコープ内で一意である必要がある。

| Agent | 説明 |
|---|---|
| _(まだなし)_ | 共有エージェントを追加するとここに一覧表示される。 |
