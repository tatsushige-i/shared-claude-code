---
name: config-claude-sync
description: Sync shared rules and skills from shared-claude-code repository - detect missing symlinks and create them
---

# Shared Config Sync Skill

Sync shared rules and skills from the shared-claude-code repository to the current project. Detect missing symlinks and create them after user confirmation.

## Steps

### Step 1: Locate the Shared Repository

1. Search for symlinks under `.claude/rules/shared/` and `.claude/skills/`
2. Resolve the shared-claude-code repository path from the `readlink` result of found symlinks
   - Rule link example: `../../../shared-claude-code/rules/conventions.md` → extract `shared-claude-code` path
   - Skill link example: `../../shared-claude-code/skills/git-pr-create` → extract `shared-claude-code` path
3. If no symlinks are found → display error and exit:
   ```
   エラー: shared-claude-codeへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. Verify that `rules/` and `skills/` directories exist at the resolved path
   - If they do not exist → display error and exit

### Step 2: Detect Differences

1. Get the list of `.md` files under `rules/` in shared-claude-code
2. Get the list of directories containing `SKILL.md` under `skills/` in shared-claude-code
3. Compare with the current project and detect missing items:
   - **Rules**: no symlink exists at `.claude/rules/shared/<name>.md`
   - **Skills**: no symlink exists at `.claude/skills/<name>`

### Step 3: Present Differences and Confirm with User

1. If everything is already synced → display the following and exit:
   ```
   すべてのルール・スキルは同期済みです。
   ```
2. If there are missing items, display in the following format:
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
3. Branch based on the user's response:
   - Full approval → sync all items
   - Partial exclusion → exclude the specified items and sync the rest

### Step 4: Create Branch

1. Check the current branch with `git branch --show-current`
2. **If not on main** → skip this step and proceed to Step 5 (assume working on an existing branch)
3. **If on main**:
   - Check for uncommitted changes with `git status --porcelain`
     - If changes exist → display error and exit:
       ```
       エラー: mainブランチに未コミットの変更があります。変更をコミットまたはスタッシュしてから再実行してください。
       ```
   - Check if a branch with the same name exists with `git branch --list chore/sync-claude-rules`
     - If it does not exist → create branch with `git checkout -b chore/sync-claude-rules`
     - If it exists → create branch with `git checkout -b chore/sync-claude-rules-YYYYMMDD` (current date)

### Step 5: Create Symlinks

1. Rule sync:
   - Create `.claude/rules/shared/` directory with `mkdir -p` if it does not exist
   - Get the prefix from the `readlink` result of existing rule symlinks and create new symlinks using the same pattern
   - Example: if an existing link is `../../../shared-claude-code/rules/conventions.md`, create new ones as `../../../shared-claude-code/rules/<name>.md`
2. Skill sync:
   - Get the prefix from the `readlink` result of existing skill symlinks and create new symlinks using the same pattern
   - Example: if an existing link is `../../shared-claude-code/skills/git-pr-create`, create new ones as `../../shared-claude-code/skills/<name>`
3. After creating each symlink, verify that the link target resolves correctly

### Step 6: Commit

1. Stage the symlinks created in Step 5 individually (do not use `git add -A` or `git add .`):
   - Rules: `git add .claude/rules/shared/<name>.md`
   - Skills: `git add .claude/skills/<name>`
2. Commit with `git commit -m "chore: sync shared claude rules and skills"`
3. If the commit fails (e.g., no staged files), display a warning and proceed to Step 7

### Step 7: Display Results

Display sync results in the following format:

```
## 同期完了

- ブランチ: <ブランチ名>（新規作成 / 既存）
- コミット: <コミットハッシュ短縮>
- 同期したルール: X件
  - <ファイル名1>
  - <ファイル名2>
- 同期したスキル: X件
  - <スキル名1>
  - <スキル名2>

`/git-pr-create` でPRを作成できます。
```

- Display "`/git-pr-create` でPRを作成できます。" only when a new branch was created in Step 4
- If running on a branch other than main, display "既存" and omit the PR suggestion
