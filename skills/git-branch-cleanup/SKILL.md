---
name: git-branch-cleanup
description: Worktree-scoped cleanup — remove the current worktree and its working branch only, leaving other worktrees and branches untouched
---

# Branch Cleanup Skill

Clean up only the **current worktree** and the **working branch checked out in it** after a PR is merged.

This skill assumes a parallel development workflow where Claude Code and Codex each operate in their own clones / worktrees and may have multiple in-flight Issues at once. Other local branches and other worktrees (e.g., Claude Code's main clone, another Issue's Codex worktree) are **never** deleted and never block this skill.

## Scope

- **Targets**: the current worktree (the one this skill is invoked from) and the branch currently checked out in it.
- **Out of scope**: every other local branch, every other linked worktree, and every other clone. They are neither inspected nor modified.
- The skill takes no arguments.

## Execution Policy

All operations are pre-approved when the user explicitly runs `/git-branch-cleanup`. Strictly follow these rules:

- **Execute all operations in a single Bash tool call command chain. Never split into multiple Bash tool calls**
- Treat `git checkout main`, `git worktree remove`, `git branch -D`, and `git fetch origin` as pre-approved operations
- Do not insert confirmation prompts like "Proceed?" or "Continue?" between steps
- Stop only when an error occurs, or when the skill detects "nothing to clean" (primary worktree on `main`)

## Execution Command

Execute the following command chain as-is in **a single Bash tool call**:

```bash
set -e
STATUS=$(git status --porcelain)
if [ -n "$STATUS" ]; then echo "エラー: 未コミットの変更があります。"; exit 1; fi
BRANCH=$(git branch --show-current)
GITDIR=$(git rev-parse --git-dir)
COMMONDIR=$(git rev-parse --git-common-dir)
if [ "$GITDIR" = "$COMMONDIR" ]; then
  if [ "$BRANCH" = "main" ]; then echo "対象なし: 主 worktree の main にいるため削除対象がありません。"; exit 0; fi
  git checkout main
  git branch -D "$BRANCH"
else
  WORKTREE_PATH=$(git rev-parse --show-toplevel)
  MAIN_PATH=$(git worktree list --porcelain | awk '/^worktree/{print $2; exit}')
  cd "$MAIN_PATH"
  git worktree remove "$WORKTREE_PATH"
  git branch -D "$BRANCH"
fi
git fetch origin
```

## Steps (Reference)

### Step 1: Check Prerequisites

1. Check for uncommitted changes in the current worktree with `git status --porcelain`
   - If changes exist → display error and exit
2. Record the working branch with `git branch --show-current`
3. Detect whether the current worktree is the **primary worktree** (main clone) or a **linked worktree** by comparing `git rev-parse --git-dir` and `git rev-parse --git-common-dir` (equal → primary, differ → linked)

### Step 2A: Primary Worktree

If the current worktree is the primary worktree (the main clone):

1. If the current branch is `main` → display `対象なし: 主 worktree の main にいるため削除対象がありません。` and exit. Do not touch any other branch or worktree.
2. Otherwise → `git checkout main`, then `git branch -D <working branch>`.

### Step 2B: Linked Worktree

If the current worktree is a linked worktree:

1. Capture the worktree path with `git rev-parse --show-toplevel`.
2. Capture the primary worktree path from the first `worktree` line of `git worktree list --porcelain`.
3. `cd` into the primary worktree.
4. `git worktree remove <linked worktree path>` to remove the worktree directory.
5. `git branch -D <working branch>` to delete the branch.
6. Other linked worktrees and their branches are left alone.

### Step 3: Refresh Remote Refs

1. Run `git fetch origin` to update remote-tracking refs.
2. The primary worktree's `main` working copy is **not** fast-forwarded automatically — the user updates it when they next switch to that worktree.

### Step 4: Display Results

Display results in the following format:

```text
## ブランチクリーンアップ完了

- 削除した worktree: <path もしくは "なし（主 worktree）">
- 削除したブランチ: <branch 名>
- git fetch origin: <結果サマリ>
```

If the skill exited at Step 2A as "対象なし", report that instead and skip the deletion lines.
