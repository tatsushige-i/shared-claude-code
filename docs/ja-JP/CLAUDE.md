# CLAUDE.md

このファイルは、このリポジトリで作業する Claude Code (claude.ai/code) へのガイダンスを提供します。

## プロジェクト概要

複数のプロジェクトへ配布する、Claude Code の共通ルールとスキルの集約リポジトリです。すべての消費リポジトリは、このリポジトリと同じ親ディレクトリに配置する必要があります。

**配布方式:**

- **ルール・スキル・エージェント** — シンボリックリンクで消費リポジトリの `.claude/rules/`・`.claude/skills/`・`.claude/agents/` に配布
- **GitHub設定とCIテンプレート** — `/config-github-sync` スキルによるファイルコピーで配布

## アーキテクチャ

- **rules/** — マスタールールファイル（英語）、消費リポジトリにシンボリックリンク
- **skills/** — マスタースキル定義、消費リポジトリにシンボリックリンク
- **agents/** — マスター subagent 定義（単一 `.md` ファイル）、消費リポジトリにシンボリックリンク
- **hooks/** — 共通hooks定義（`shared-hooks.json`）、`config-claude-sync` でマージ
- **.claude/** — `rules/`・`skills/`・`agents/` へのシンボリックリンク、およびこのリポジトリ用 `settings.json`
- **ci-templates/** — 言語別CIテンプレート、`/config-github-sync` でコピー
- **.github/** — Issue/PRテンプレート、`/config-github-sync` でコピー
- **docs/ja-JP/** — 日本語翻訳（補足資料。英語版が正）

## ドキュメントの記述言語（このリポジトリ）

このリポジトリのマスター `.md` ファイル（rules、skills、agents、README、CLAUDE.md）はすべて **英語** で記述する。日本語版は補足資料として `docs/ja-JP/` に配置し、`docs/ja-JP/` は英語マスタの翻訳専用とする。

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

## 新しいAgentの追加手順

subagent は、YAML フロントマター（`name`, `description`、任意で `tools`, `model`）とシステムプロンプト本文からなる単一の Markdown ファイル。起動時の識別子はファイル名ではなく `name` フィールドで決まり、スコープ内で一意である必要がある。`model` フィールドでモデルを固定できる（`opus`, `sonnet`, `haiku`, `fable`, `inherit`）。

エージェント追加時、**共有エージェント**か**ローカルエージェント**かをユーザーに確認すること。

### 共有エージェント（シンボリックリンク経由で消費リポジトリに配布）

1. `agents/<name>.md` を YAML フロントマター（`name`, `description`）付きで作成
2. シンボリックリンクを追加: `.claude/agents/<name>.md -> ../../agents/<name>.md`
3. `agents/README.md` のエージェント一覧テーブルに行を追加
4. `docs/ja-JP/agents/<name>.md` に日本語翻訳を追加

消費リポジトリでは、共有エージェントは `.claude/agents/shared/` 配下に同期される（Claude Code は `.claude/agents/` を再帰的に走査するため、サブディレクトリは起動に影響しない）。

### ローカルエージェント（このリポジトリ専用）

エージェント名の先頭に `local-` を付与する（例: `local-doc-reviewer`）。

1. `.claude/agents/local-<name>.md` を YAML フロントマター（`name`, `description`）付きで直接作成（シンボリックリンク不要）
2. `docs/ja-JP/local-agents/local-<name>.md` に日本語翻訳を追加

## 新しいHookの追加手順

### 共通Hook（`config-claude-sync` で消費リポジトリに配布）

**判断基準**: プロジェクト非依存で、どのプロジェクトでも汎用的に使えるhook。

1. `hooks/shared-hooks.json` にエントリを追加する（`settings.json` の `hooks` 構造に準拠し、`_id`・`_description`・`_detect_by` メタデータフィールドを含める）
2. このリポジトリの `.claude/settings.json` にも同じhookを追加する

### リポジトリ固有Hook（このリポジトリのみで使用）

**判断基準**: このリポジトリのファイル構成・ツール・ワークフローに固有のhook。

1. `.claude/settings.json` にのみ追加する
