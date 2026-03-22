---
name: config-github-sync
description: Sync .github files (ISSUE_TEMPLATE, workflows) from shared-claude-code repository - detect diffs and copy files
---

# GitHub Config Sync Skill

shared-claude-codeリポジトリの `.github/` 配下の共有資材（ISSUE_TEMPLATE、workflows）を現在のプロジェクトにコピーベースで同期する。GitHubがシンボリックリンクを認識しないため、ファイルコピーで同期する。

## 処理手順

### Step 1: 共通リポジトリの特定

1. `.claude/rules/shared/` および `.claude/skills/` 配下のシンボリックリンクを探索する
2. 見つかったシンボリックリンクの `readlink` 結果からshared-claude-codeリポジトリのパスを解決する
   - ルールのリンク例: `../../../shared-claude-code/rules/conventions.md` → `shared-claude-code` のパスを抽出
   - スキルのリンク例: `../../shared-claude-code/skills/git-pr-create` → `shared-claude-code` のパスを抽出
3. シンボリックリンクが1つも見つからない場合 → エラーを表示して終了する:
   ```
   エラー: shared-claude-codeへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. 解決したパスに `.github/` ディレクトリが存在することを検証する
   - 存在しない場合 → エラーを表示して終了する

### Step 2: 差分の検出

1. shared側の `.github/ISSUE_TEMPLATE/` と `.github/workflows/` を走査する
2. ローカルの対応ファイルと比較し、各ファイルの状態を判定する:
   - **同一**: 内容が一致（`diff -q` で判定）
   - **差分あり**: 両方に存在するが内容が異なる
   - **新規**: shared側にあるがローカルに存在しない
   - **ローカルのみ**: ローカルにあるがshared側に存在しない（警告表示のみ、削除はしない）

### Step 3: 差分の提示・ユーザー確認

1. すべて同一の場合 → 以下を表示して終了する:
   ```
   .github/ 配下のファイルはすべて同期済みです。
   ```
2. 差分がある場合、カテゴリ別に状態を一覧表示する:
   ```
   ## .github 同期チェック

   ### ISSUE_TEMPLATE
   - bug.yml — 同一 ✓
   - chore.yml — ローカルに存在しない（新規追加）
   - enhancement.yml — 差分あり

   ### workflows
   - ci.yml — 差分あり

   同期する項目を選択してください（例: 全て / ISSUE_TEMPLATEのみ）。
   差分の詳細を確認したい場合はお知らせください。
   ```
3. ユーザーが差分詳細を要求した場合 → `diff` 出力を表示する
4. ユーザーの回答に応じて同期対象を決定する:
   - 全承認 → 全項目を同期する
   - 一部除外を指示 → 指示されたものを除外して同期する

### Step 4: ファイルコピー

1. `.github/ISSUE_TEMPLATE/` や `.github/workflows/` ディレクトリが存在しない場合は `mkdir -p` で作成する
2. 選択された項目を `cp` で上書きコピーする
3. コピー後、`diff -q` で内容が一致することを検証する
   - 不一致の場合 → エラーを表示する

### Step 5: 結果表示

同期結果を以下の形式で表示する:

```
## 同期完了

- 同期したISSUE_TEMPLATE: X件
  - <ファイル名1>（新規追加）
  - <ファイル名2>（更新）
- 同期したワークフロー: X件
  - <ファイル名1>（更新）
```
