# shared-claude-code

[English](../../README.md)

シンボリックリンクを使い、複数リポジトリで共有するClaude Codeのルールとスキル。

## 構成

```
rules/
├── conventions.md          # 開発規約
└── security.md             # セキュリティルール
skills/
├── config-claude-sync/     # shared-claude-codeからルール・スキルをシンボリックリンクで同期
├── config-github-sync/     # shared-claude-codeから.githubファイルをコピーで同期
├── git-branch-cleanup/     # PRマージ後のローカルブランチクリーンアップ
├── git-issue-create/       # 会話コンテキストからGitHub Issueを作成
├── git-issue-start/        # GitHub Issueからの作業開始ワークフロー
├── git-pr-create/          # 分析付きGitHub PR作成
└── git-review-respond/     # PRレビューコメントへの対応
```

## 使い方

各リポジトリから `.claude/rules/` と `.claude/skills/` にシンボリックリンクを作成する:

```bash
# 対象リポジトリのルートから実行
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

**前提条件**: すべてのリポジトリが同じ親ディレクトリに配置されていること。
