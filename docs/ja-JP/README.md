**Language:** [English](../../README.md) | 日本語

# shared-claude-code

複数リポジトリで共有する Claude Code のルールとスキルの集約リポジトリ。ルールとスキルは**シンボリックリンク**で、GitHub設定とCIテンプレートは**ファイルコピー**で配布する。

## 構成

```
rules/                  # マスタールールファイル（英語） — シンボリックリンクで消費リポジトリに配布
├── conventions.md      # 開発規約
├── security.md         # セキュリティルール
└── tech-debt-checklist.md # 技術的負債チェックリスト
skills/                 # マスタースキル定義 — シンボリックリンクで消費リポジトリに配布
├── README.md           # スキル一覧テーブル
├── config-claude-sync/ # ルール・スキルをシンボリックリンクで同期
├── config-github-sync/ # .githubファイルとCIテンプレートをファイルコピーで同期
├── git-branch-cleanup/ # PRマージ後のローカルブランチクリーンアップ
├── git-issue-create/   # 会話コンテキストからGitHub Issueを作成
├── git-issue-start/    # GitHub Issueからの作業開始ワークフロー
├── git-pr-create/      # 分析付きGitHub PR作成
├── git-review-respond/ # PRレビューコメントへの対応
└── tech-debt-audit-nextjs/ # Next.jsプロジェクトの技術的負債調査
ci-templates/           # 言語別CI/設定テンプレート — ファイルコピーで消費リポジトリに配布
└── nextjs/             # Next.js テンプレート（ESLint, Jest, TypeScript設定）
.github/
└── ISSUE_TEMPLATE/     # Issueテンプレート（日本語） — ファイルコピーで消費リポジトリに配布
docs/ja-JP/             # 日本語翻訳（補足資料。英語版が正）
.claude/
├── rules/              # シンボリックリンク → ../../rules/
└── skills/             # シンボリックリンク → ../../skills/ + ローカルスキル
```

## セットアップ手順

**前提条件**: すべての消費リポジトリがこのリポジトリと同じ親ディレクトリに配置されていること。

### 1. ルールとスキルのシンボリックリンク作成

消費リポジトリのルートから、共有ルールとスキルへのシンボリックリンクを作成する:

```bash
# ルール
ln -s ../../shared-claude-code/rules/conventions.md .claude/rules/shared/conventions.md
ln -s ../../shared-claude-code/rules/security.md .claude/rules/shared/security.md
ln -s ../../shared-claude-code/rules/tech-debt-checklist.md .claude/rules/shared/tech-debt-checklist.md

# スキル
ln -s ../../shared-claude-code/skills/config-claude-sync .claude/skills/config-claude-sync
ln -s ../../shared-claude-code/skills/config-github-sync .claude/skills/config-github-sync
ln -s ../../shared-claude-code/skills/git-branch-cleanup .claude/skills/git-branch-cleanup
ln -s ../../shared-claude-code/skills/git-issue-create .claude/skills/git-issue-create
ln -s ../../shared-claude-code/skills/git-issue-start .claude/skills/git-issue-start
ln -s ../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
ln -s ../../shared-claude-code/skills/git-review-respond .claude/skills/git-review-respond
ln -s ../../shared-claude-code/skills/tech-debt-audit-nextjs .claude/skills/tech-debt-audit-nextjs
```

または `/config-claude-sync` スキルを使って、不足しているシンボリックリンクを自動検出・作成できる。

### 2. GitHub設定とCIテンプレートのコピー

`/config-github-sync` スキルを使って、Issueテンプレート、ワークフローファイル、CI設定テンプレートをリポジトリにコピーする。

## 利用可能なスキル

| スキル | コマンド | 説明 |
|---|---|---|
| `config-claude-sync` | `/config-claude-sync` | `.claude/`配下（rules, skills）の差分検出・シンボリックリンク同期 |
| `config-github-sync` | `/config-github-sync` | `.github/`配下（ISSUE_TEMPLATE, workflows）の差分検出・コピー同期 |
| `git-branch-cleanup` | `/git-branch-cleanup` | ローカルブランチクリーンアップ（main切替・ブランチ削除・pull） |
| `git-issue-create` | `/git-issue-create` | 会話の文脈からIssue作成（タイトル・本文・ラベル推定・プレビュー） |
| `git-issue-start` | `/git-issue-start <Issue#>` | Issue取得・ラベル検証・ブランチ作成・Plan Mode移行 |
| `git-pr-create` | `/git-pr-create` | Issue特定・規模チェック・差分分析・PR作成 |
| `git-review-respond` | `/git-review-respond <PR#>` | レビューコメント分析・コード修正・返信 |
| `tech-debt-audit-nextjs` | `/tech-debt-audit-nextjs` | Next.js（App Router）プロジェクトの技術的負債調査・優先度付きレポート生成 |
