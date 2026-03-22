# Skills

シンボリックリンク経由で消費リポジトリに配布される共有スキル一覧。各スキルは Claude Code でスラッシュコマンドとして呼び出せる。`/config-claude-sync` を使えば不足しているシンボリックリンクを自動検出・同期できる。

| スキル | コマンド | 説明 |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | `.claude/`配下（rules, skills）の差分検出・シンボリックリンク同期 |
| `config-github-sync` | `/config-github-sync` | `.github/`配下（ISSUE_TEMPLATE, workflows）の差分検出・コピー同期 |
| `git-branch-cleanup` | `/git-branch-cleanup` | ローカルブランチクリーンアップ（main切替・ブランチ削除・pull） |
| `git-issue-create` | `/git-issue-create` | 会話の文脈からIssue作成（タイトル・本文・ラベル推定・プレビュー） |
| `git-issue-start` | `/git-issue-start <Issue#>` | Issue取得・ラベル検証・ブランチ作成・Plan Mode移行 |
| `git-pr-create` | `/git-pr-create` | Issue特定・規模チェック・差分分析・PR作成 |
| `git-review-respond` | `/git-review-respond <PR#>` | レビューコメント分析・コード修正・返信 |
