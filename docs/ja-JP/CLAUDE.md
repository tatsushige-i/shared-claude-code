# CLAUDE.md

このファイルは、このリポジトリで作業する Claude Code (claude.ai/code) へのガイダンスを提供します。

## このリポジトリについて

複数のプロジェクトへ配布する、Claude Code の共通ルールとスキルの集約リポジトリです。すべての消費リポジトリは、このリポジトリと同じ親ディレクトリに配置する必要があります。

**配布方式:**
- **ルールとスキル** — シンボリックリンクで消費リポジトリの `.claude/rules/` と `.claude/skills/` に配布
- **GitHub設定とCIテンプレート** — `/config-github-sync` スキルによるファイルコピーで配布

**シンボリックリンクの作成例**（消費リポジトリ側で実行）:
```bash
ln -s ../../shared-claude-code/rules/conventions.md .claude/rules/conventions.md
ln -s ../../shared-claude-code/skills/git-pr-create .claude/skills/git-pr-create
```

## ディレクトリ構成

```
rules/          # マスタールールファイル（英語） — 消費リポジトリの .claude/rules/ にシンボリックリンク
skills/         # マスタースキル定義 — 消費リポジトリの .claude/skills/ にシンボリックリンク
  README.md     # スキル一覧テーブル — スキル追加時に更新必須
.claude/
  rules/        # シンボリックリンク → ../../rules/
  skills/       # シンボリックリンク → ../../skills/
ci-templates/   # 言語別CIテンプレート — /config-github-sync によるファイルコピーで消費リポジトリに配布
  nextjs/       # Next.js テンプレート（ESLint, Jest, TypeScript設定）
.github/
  ISSUE_TEMPLATE/   # Issueテンプレート（日本語） — /config-github-sync によるファイルコピーで消費リポジトリに配布
docs/ja-JP/     # 日本語翻訳（補足資料。英語版が正）
  rules/        # rules/ の翻訳
  skills/       # skills/ の翻訳
  local-skills/ # ローカルスキルの翻訳
```

## 新しいスキルの追加手順

スキル追加時、**共有スキル**か**ローカルスキル**かをユーザーに確認すること。

### 共有スキル（シンボリックリンク経由で消費リポジトリに配布）

1. `skills/<name>/SKILL.md` を YAML フロントマター（`name`, `description`）付きで作成
2. シンボリックリンクを追加: `.claude/skills/<name> -> ../../skills/<name>`
3. `skills/README.md` のスキル一覧テーブルに行を追加
4. `docs/ja-JP/skills/<name>/SKILL.md` に日本語翻訳を追加

### ローカルスキル（このリポジトリ専用）

スキル名の先頭に `local-` を付与する（例: `local-docs-validate`）。

1. `.claude/skills/local-<name>/SKILL.md` を YAML フロントマター（`name`, `description`）付きで直接作成（シンボリックリンク不要）
2. `docs/ja-JP/local-skills/local-<name>/SKILL.md` に日本語翻訳を追加

## 新しいルールの追加手順

### 共有ルール（シンボリックリンク経由で消費リポジトリに配布）

1. `rules/<name>.md` を作成
2. シンボリックリンクを追加: `.claude/rules/<name>.md -> ../../rules/<name>.md`
3. `docs/ja-JP/rules/<name>.md` に日本語翻訳を追加
