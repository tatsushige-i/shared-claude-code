# Agents

シンボリックリンクで消費リポジトリに配布する、共有 subagent（エージェント）定義。各エージェントは YAML フロントマター（`name`, `description`、任意で `tools`, `model`）とシステムプロンプト本文からなる単一の Markdown ファイル（`agents/<name>.md`）。エージェントは Agent ツールの `subagent_type` として（スキル内からも）起動できる。不足しているシンボリックリンクは `/config-claude-sync` で自動検出・同期できる。

消費リポジトリでは、共有エージェントは `.claude/agents/shared/` 配下にシンボリックリンクされる。Claude Code は `.claude/agents/` を再帰的に走査するため、`shared/` サブディレクトリはエージェントの識別・起動方法に影響しない。識別子はフロントマターの `name` フィールドのみで決まり、スコープ内で一意である必要がある。

| Agent | 説明 |
|---|---|
| `review-correctness` | 正確性・バグに特化したコードレビュアー（opus）。`review-team-run` の並列チームの一員。 |
| `review-security` | セキュリティに特化したコードレビュアー（opus）。`review-team-run` の並列チームの一員。 |
| `review-design` | 設計・シンプルさに特化したコードレビュアー（opus）。`review-team-run` の並列チームの一員。 |
| `review-readability` | 可読性・クリーンさに特化したコードレビュアー（sonnet）。`review-team-run` の並列チームの一員。 |
| `review-tests` | テストに特化したコードレビュアー（sonnet）。`review-team-run` の並列チームの一員。 |
| `impl-planner` | 読み取り専用の実装プランナー（opus）。Issue から影響ファイル・手順・判断ポイントを洗い出す。`impl-pipeline` スキルから使う。 |
| `impl-coder` | 自律実装担当（sonnet）。機械的な Issue で確定プランを実行する。`impl-pipeline` スキルから使う。 |
| `impl-tester` | テスト担当（sonnet）。実装ではなく仕様からテストを書いて実行する。`impl-pipeline` スキルから使う。 |
