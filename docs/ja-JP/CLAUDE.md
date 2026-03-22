# CLAUDE.md

このファイルは、このリポジトリで作業する Claude Code (claude.ai/code) へのガイダンスを提供します。

## このリポジトリについて

シンボリックリンクを通じて複数のプロジェクトへ配布する、Claude Code の共通ルールとスキルの集約リポジトリです。すべての消費リポジトリは、このリポジトリと同じ親ディレクトリに配置する必要があります。

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
.github/
  ISSUE_TEMPLATE/   # Issue テンプレート 5種（日本語）
  workflows/ci.yml  # CI: eol-check → lint → test → build
docs/ja-JP/     # 日本語翻訳（補足資料。英語版が正）
.claude/
  rules/        # シンボリックリンク → ../../rules/
  skills/       # シンボリックリンク → ../../skills/
```

## 新しいスキルの追加手順

1. `skills/<category>-<object>-<verb>/SKILL.md` を YAML フロントマター（`name`, `description`）付きで作成
2. シンボリックリンクを追加: `.claude/skills/<name> -> ../../skills/<name>`
3. `skills/README.md` のスキル一覧テーブルに行を追加
4. `docs/ja-JP/skills/<name>/SKILL.md` に日本語翻訳を追加
