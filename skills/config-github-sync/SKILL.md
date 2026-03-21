---
name: config-github-sync
description: Sync .github files (ISSUE_TEMPLATE, workflows) from shared-claude-rules repository - detect diffs and copy files
---

# GitHub Config Sync Skill

Sync shared assets under `.github/` (ISSUE_TEMPLATE, workflows) from the shared-claude-rules repository to the current project using file copies. File copies are used because GitHub does not recognize symlinks.

## Steps

### Step 1: Locate the Shared Repository

1. Search for symlinks under `.claude/rules/shared/` and `.claude/skills/`
2. Resolve the shared-claude-rules repository path from the `readlink` result of found symlinks
   - Rule link example: `../../../shared-claude-rules/rules/conventions.md` → extract `shared-claude-rules` path
   - Skill link example: `../../shared-claude-rules/skills/git-pr-create` → extract `shared-claude-rules` path
3. If no symlinks are found → display error and exit:
   ```
   エラー: shared-claude-rulesへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. Verify that the `.github/` directory exists at the resolved path
   - If it does not exist → display error and exit

### Step 2: Detect Differences

1. Scan `.github/ISSUE_TEMPLATE/` and `.github/workflows/` on the shared side
2. Compare with the corresponding local files and determine each file's status:
   - **Identical**: contents match (determined by `diff -q`)
   - **Differs**: exists on both sides but contents differ
   - **New**: exists on the shared side but not locally
   - **Local only**: exists locally but not on the shared side (display warning only, do not delete)

### Step 3: Present Differences and Confirm with User

1. If all files are identical → display the following and exit:
   ```
   .github/ 配下のファイルはすべて同期済みです。
   ```
2. If differences exist, display status by category:
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
3. If the user requests diff details → show the `diff` output
4. Determine sync targets based on the user's response:
   - Full approval → sync all items
   - Partial exclusion → exclude the specified items and sync the rest

### Step 4: Copy Files

1. Create `.github/ISSUE_TEMPLATE/` and `.github/workflows/` directories with `mkdir -p` if they do not exist
2. Overwrite-copy selected items with `cp`
3. After copying, verify contents match with `diff -q`
   - If mismatch → display error

### Step 5: Display Results

Display sync results in the following format:

```
## 同期完了

- 同期したISSUE_TEMPLATE: X件
  - <ファイル名1>（新規追加）
  - <ファイル名2>（更新）
- 同期したワークフロー: X件
  - <ファイル名1>（更新）
```
