---
name: config-shared-sync
description: Sync shared rules and skills from shared-claude-rules repository - detect missing symlinks and create them
---

# Shared Config Sync Skill

shared-claude-rulesリポジトリの共通ルール・スキルを現在のプロジェクトに同期する。未取り込みのシンボリックリンクを検出し、ユーザー確認の上で作成する。

## 処理手順

### Step 1: 共通リポジトリの特定

1. `.claude/rules/shared/` および `.claude/skills/` 配下のシンボリックリンクを探索する
2. 見つかったシンボリックリンクの `readlink` 結果からshared-claude-rulesリポジトリのパスを解決する
   - ルールのリンク例: `../../../shared-claude-rules/rules/conventions.md` → `shared-claude-rules` のパスを抽出
   - スキルのリンク例: `../../shared-claude-rules/skills/git-pr-create` → `shared-claude-rules` のパスを抽出
3. シンボリックリンクが1つも見つからない場合 → エラーを表示して終了する:
   ```
   エラー: shared-claude-rulesへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. 解決したパスに `rules/` と `skills/` ディレクトリが存在することを検証する
   - 存在しない場合 → エラーを表示して終了する

### Step 2: 差分の検出

1. shared-claude-rulesの `rules/` 配下の `.md` ファイル一覧を取得する
2. shared-claude-rulesの `skills/` 配下で `SKILL.md` を含むディレクトリ一覧を取得する
3. 現在のプロジェクトと比較し、未取り込みのものを検出する:
   - **ルール**: `.claude/rules/shared/<name>.md` にシンボリックリンクが存在しないもの
   - **スキル**: `.claude/skills/<name>` にシンボリックリンクが存在しないもの

### Step 3: 差分の提示・ユーザー確認

1. すべて同期済みの場合 → 以下を表示して終了する:
   ```
   すべてのルール・スキルは同期済みです。
   ```
2. 未取り込みがある場合、以下の形式で表示する:
   ```
   ## 未同期の項目

   ### ルール
   - conventions.md
   - security.md

   ### スキル
   - git-branch-cleanup
   - git-issue-create

   上記の項目を同期しますか？除外したい項目があればお知らせください。
   ```
3. ユーザーの回答に応じて分岐する:
   - 全承認 → 全項目を同期する
   - 一部除外を指示 → 指示されたものを除外して同期する

### Step 4: シンボリックリンク作成

1. ルールの同期:
   - `.claude/rules/shared/` ディレクトリが存在しない場合は `mkdir -p` で作成する
   - 既存のルールシンボリックリンクの `readlink` 結果からプレフィックスを取得し、同じパターンで新しいシンボリックリンクを作成する
   - 例: 既存リンクが `../../../shared-claude-rules/rules/conventions.md` なら、新規も `../../../shared-claude-rules/rules/<name>.md` で作成
2. スキルの同期:
   - 既存のスキルシンボリックリンクの `readlink` 結果からプレフィックスを取得し、同じパターンで新しいシンボリックリンクを作成する
   - 例: 既存リンクが `../../shared-claude-rules/skills/git-pr-create` なら、新規も `../../shared-claude-rules/skills/<name>` で作成
3. 各シンボリックリンク作成後、リンク先が正しく解決できることを確認する

### Step 5: 結果表示

同期結果を以下の形式で表示する:

```
## 同期完了

- 同期したルール: X件
  - <ファイル名1>
  - <ファイル名2>
- 同期したスキル: X件
  - <スキル名1>
  - <スキル名2>
```
