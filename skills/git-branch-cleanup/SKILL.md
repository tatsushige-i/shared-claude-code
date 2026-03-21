---
name: git-branch-cleanup
description: Clean up local branches - switch to main, delete all local branches, and pull latest
---

# Branch Cleanup Skill

Automate local branch cleanup after PR creation/merge. Switch to main, delete local branches, and pull latest in a single flow.

## Execution Policy

All operations are pre-approved when the user explicitly runs `/git-branch-cleanup`. Strictly follow these rules:

- **Execute all operations in a single Bash tool call command chain. Never split into multiple Bash tool calls**
- Treat `git checkout main`, `git branch -D`, and `git pull` as pre-approved operations
- Do not insert confirmation prompts like "Proceed?" or "Continue?" between steps
- Stop only when an error occurs

## Execution Command

Execute the following command chain as-is in **a single Bash tool call**:

```bash
set -e; STATUS=$(git status --porcelain); [ -n "$STATUS" ] && echo "エラー: 未コミットの変更があります。" && exit 1; git checkout main 2>&1; git branch | grep -v '^\*' | xargs -r git branch -D 2>&1; git pull 2>&1
```

## Steps (Reference)

### Step 1: Check Prerequisites

1. Check for uncommitted changes with `git status --porcelain`
   - If changes exist → display error and exit
2. Get current branch with `git branch --show-current`
   - If already on `main` → skip branch switch (Step 2)

### Step 2: Switch to main Branch

1. Run `git checkout main`
2. If it fails, display error and exit

### Step 3: Delete Local Branches

1. Get list of local branches other than main with `git branch`
   - If no branches exist → display "No branches to delete" and proceed to Step 4
2. Delete each branch with `git branch -D <branch-name>`

### Step 4: Pull Latest

1. Run `git pull`
2. If it fails, display error

### Step 5: Display Results

Display results in the following format:

```
## ブランチクリーンアップ完了

- 現在のブランチ: main
- 削除したブランチ: X件
  - <ブランチ名1>
  - <ブランチ名2>
- git pull: <結果サマリ>
```
