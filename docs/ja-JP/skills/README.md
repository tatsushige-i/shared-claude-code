# Skills

シンボリックリンク経由で消費リポジトリに配布される共有スキル一覧。各スキルは Claude Code でスラッシュコマンドとして呼び出せる。`/config-claude-sync` を使えば不足しているシンボリックリンクを自動検出・同期できる。

| スキル | コマンド | 説明 |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | `.claude/`配下（rules, skills）の差分検出・シンボリックリンク同期 |
| `config-github-sync` | `/config-github-sync` | `.github/`配下（ISSUE_TEMPLATE, workflows）の差分検出・コピー同期 |
| `git-branch-cleanup` | `/git-branch-cleanup` | 対象 worktree とその作業ブランチに限定したクリーンアップ |
| `git-issue-create` | `/git-issue-create` | 会話の文脈からIssue作成（タイトル・本文・ラベル推定・プレビュー） |
| `git-issue-start` | `/git-issue-start <Issue#>` | Issue取得・ラベル検証・ブランチ作成・Plan Mode移行 |
| `git-pr-create` | `/git-pr-create` | Issue特定・規模チェック・差分分析・PR作成 |
| `git-review-respond` | `/git-review-respond <PR#>` | レビューコメント分析・コード修正・返信 |
| `impl-pipeline` | `/impl-pipeline <Issue#>` | 支援エージェント群（plan/code/test）で Issue 実装を進め、人間ゲートで機械的 vs. 対話を切り分け、review-team-run を再利用し、git-pr-create へ引き継ぐ |
| `review-team-run` | `/review-team-run` | スタンスの異なるレビュー subagent を PR作成前の差分に並列起動し、統合レポートを提示 |
