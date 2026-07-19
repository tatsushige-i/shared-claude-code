**Language:** [English](../../README.md) | 日本語

# shared-claude-code

複数リポジトリで共有する Claude Code のルール・スキル・エージェントの集約リポジトリ。ルール・スキル・エージェントは**シンボリックリンク**で、GitHub設定とCIテンプレートは**ファイルコピー**で配布する。

## 構成

```text
rules/                  # マスタールールファイル（英語） — シンボリックリンクで消費リポジトリに配布
├── conventions.md      # 開発規約
├── security.md         # セキュリティルール
└── tech-debt-checklist.md # 技術的負債チェックリスト
skills/                 # マスタースキル定義 — シンボリックリンクで消費リポジトリに配布
├── README.md           # スキル一覧テーブル
├── config-claude-sync/ # ルール・スキルをシンボリックリンクで同期
├── config-github-sync/ # .githubファイルとCIテンプレートをファイルコピーで同期
├── git-branch-cleanup/ # PRマージ後の対象 worktree 限定クリーンアップ
├── git-issue-create/   # 会話コンテキストからGitHub Issueを作成
├── git-issue-start/    # GitHub Issueからの作業開始ワークフロー
├── git-pr-create/      # 分析付きGitHub PR作成
├── git-pr-finalize/    # CI/レビュー監視・指摘対応・マージ・整理
├── git-review-respond/ # PRレビューコメントへの対応
├── impl-pipeline/      # 支援エージェント群（plan/code/test）で Issue 実装を進める
├── review-team-run/    # PR作成前の差分にレビューチームを並列起動
├── tech-debt-audit-flutter/ # Flutter / Dartプロジェクトの技術的負債調査
└── tech-debt-audit-nextjs/ # Next.jsプロジェクトの技術的負債調査
agents/                 # マスター subagent 定義（単一 .md ファイル） — シンボリックリンクで消費リポジトリに配布
├── README.md           # エージェント一覧テーブル
├── impl-*.md           # Issue 実装パイプラインのエージェント（planner / coder / tester）
└── review-*.md         # 並列コードレビューチーム（正確性 / セキュリティ / 設計 / 可読性 / テスト）
hooks/                  # 共通hooks定義 — config-claude-sync で消費リポジトリの settings.json にマージ
└── shared-hooks.json   # PreToolUse / Stop / UserPromptSubmit 等の共通hooks
ci-templates/           # 言語別CI/設定テンプレート — ファイルコピーで消費リポジトリに配布
└── nextjs/             # Next.js テンプレート（ESLint, Jest, TypeScript設定）
.github/
└── ISSUE_TEMPLATE/     # Issueテンプレート（日本語） — ファイルコピーで消費リポジトリに配布
docs/ja-JP/             # 日本語翻訳（補足資料。英語版が正）
.claude/
├── rules/              # シンボリックリンク → ../../rules/
├── skills/             # シンボリックリンク → ../../skills/ + ローカルスキル
└── agents/             # シンボリックリンク → ../../agents/ + ローカルエージェント
```

## セットアップ手順

**前提条件**: すべての消費リポジトリがこのリポジトリと同じ親ディレクトリに配置されていること。

### 1. ルール・スキル・エージェントのシンボリックリンク作成

消費リポジトリのルートから、共有ルール・スキル・エージェントへのシンボリックリンクを作成する:

```bash
# 配置先ディレクトリを作成
mkdir -p .claude/rules/shared .claude/skills .claude/agents/shared

# ルール
ln -s ../../../../shared-claude-code/rules/conventions.md .claude/rules/shared/conventions.md
ln -s ../../../../shared-claude-code/rules/security.md .claude/rules/shared/security.md
ln -s ../../../../shared-claude-code/rules/tech-debt-checklist.md .claude/rules/shared/tech-debt-checklist.md

# スキル
ln -s ../../../shared-claude-code/skills/config-claude-sync .claude/skills/config-claude-sync
ln -s ../../../shared-claude-code/skills/config-github-sync .claude/skills/config-github-sync
ln -s ../../../shared-claude-code/skills/git-branch-cleanup .claude/skills/git-branch-cleanup
ln -s ../../../shared-claude-code/skills/git-issue-create .claude/skills/git-issue-create
ln -s ../../../shared-claude-code/skills/git-issue-start .claude/skills/git-issue-start
ln -s ../../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
ln -s ../../../shared-claude-code/skills/git-pr-finalize .claude/skills/git-pr-finalize
ln -s ../../../shared-claude-code/skills/git-review-respond .claude/skills/git-review-respond
ln -s ../../../shared-claude-code/skills/impl-pipeline .claude/skills/impl-pipeline
ln -s ../../../shared-claude-code/skills/review-team-run .claude/skills/review-team-run
ln -s ../../../shared-claude-code/skills/tech-debt-audit-flutter .claude/skills/tech-debt-audit-flutter
ln -s ../../../shared-claude-code/skills/tech-debt-audit-nextjs .claude/skills/tech-debt-audit-nextjs

# エージェント（.claude/agents/shared/ 配下にシンボリックリンク）
ln -s ../../../../shared-claude-code/agents/impl-planner.md .claude/agents/shared/impl-planner.md
ln -s ../../../../shared-claude-code/agents/impl-coder.md .claude/agents/shared/impl-coder.md
ln -s ../../../../shared-claude-code/agents/impl-tester.md .claude/agents/shared/impl-tester.md
ln -s ../../../../shared-claude-code/agents/review-correctness.md .claude/agents/shared/review-correctness.md
ln -s ../../../../shared-claude-code/agents/review-security.md .claude/agents/shared/review-security.md
ln -s ../../../../shared-claude-code/agents/review-design.md .claude/agents/shared/review-design.md
ln -s ../../../../shared-claude-code/agents/review-readability.md .claude/agents/shared/review-readability.md
ln -s ../../../../shared-claude-code/agents/review-tests.md .claude/agents/shared/review-tests.md
```

または `/config-claude-sync` スキルを使って、不足しているシンボリックリンクを自動検出・作成できる。

`/config-claude-sync` スキルは、`hooks/shared-hooks.json` の共通hooksエントリを消費リポジトリの `.claude/settings.json` にマージする処理も併せて行う（既存エントリは保持され、新規・更新のあったhookのみが適用される）。hooksにシンボリックリンクは不要。

### 2. GitHub設定とCIテンプレートのコピー

`/config-github-sync` スキルを使って、Issueテンプレート、ワークフローファイル、CI設定テンプレートをリポジトリにコピーする。

共通ワークフローには `close-linked-issues-on-develop.yml` が含まれる。`develop` 向けの PR がマージされたとき、PR 本文の `Closes #N` 等から紐づく Issue を自動クローズする（GitHub ネイティブの自動クローズがデフォルトブランチ以外では発火しない問題を補う）。`develop` ブランチを使わないリポジトリでは何も起こらない。

## 利用可能なスキル

| スキル | コマンド | 説明 |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | `.claude/`配下（rules, skills）の差分検出・シンボリックリンク同期、および共通hooksの `settings.json` へのマージ |
| `config-github-sync` | `/config-github-sync` | `.github/`配下（ISSUE_TEMPLATE, workflows）の差分検出・コピー同期 |
| `git-branch-cleanup` | `/git-branch-cleanup` | 対象 worktree とその作業ブランチに限定したクリーンアップ |
| `git-issue-create` | `/git-issue-create` | 会話の文脈からIssue作成（タイトル・本文・ラベル推定・プレビュー） |
| `git-issue-start` | `/git-issue-start <Issue#>` | Issue取得・ラベル検証・ブランチ作成・Plan Mode移行 |
| `git-pr-create` | `/git-pr-create` | Issue特定・規模チェック・差分分析・PR作成 |
| `git-pr-finalize` | `/git-pr-finalize [PR#]` | CI/Copilotレビュー監視・指摘対応・確認後マージ・ブランチ整理 |
| `git-review-respond` | `/git-review-respond <PR#>` | レビューコメント分析・コード修正・返信 |
| `impl-pipeline` | `/impl-pipeline <Issue#>` | 支援エージェント群（plan/code/test）で Issue 実装を進め、人間ゲートで機械的 vs. 対話を切り分け、review-team-run を再利用し、git-pr-create へ引き継ぐ |
| `review-team-run` | `/review-team-run` | スタンスの異なるレビュー subagent を PR作成前の差分に並列起動し、統合レポートを提示 |
| `tech-debt-audit-flutter` | `/tech-debt-audit-flutter` | Flutter / Dartプロジェクトの技術的負債調査・優先度付きレポート生成 |
| `tech-debt-audit-nextjs` | `/tech-debt-audit-nextjs` | Next.js（App Router）プロジェクトの技術的負債調査・優先度付きレポート生成 |
