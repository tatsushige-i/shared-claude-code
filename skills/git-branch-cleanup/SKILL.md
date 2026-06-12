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
- Treat `gh pr view`, `git checkout <home>`, `git worktree remove`, `git branch -D`, `git fetch origin`, and `git pull --ff-only origin <home>` as pre-approved operations
- Do not insert confirmation prompts like "Proceed?" or "Continue?" between steps
- Stop only when an error occurs, or when the skill detects "nothing to clean" (current branch is a protected `main`/`develop`/`release/*` branch)

## Execution Command

Execute the following command chain as-is in **a single Bash tool call**:

```bash
set -e
STATUS=$(git status --porcelain)
if [ -n "$STATUS" ]; then echo "エラー: 未コミットの変更があります。"; exit 1; fi
BRANCH=$(git branch --show-current)
case "$BRANCH" in
  main|develop|release/*)
    echo "対象なし: $BRANCH は保護ブランチのため削除対象がありません。"; exit 0 ;;
esac
HOME=$(gh pr view "$BRANCH" --json baseRefName -q .baseRefName 2>/dev/null || true)
if [ -z "$HOME" ]; then HOME=main; fi
GITDIR=$(git rev-parse --git-dir)
COMMONDIR=$(git rev-parse --git-common-dir)
if [ "$GITDIR" = "$COMMONDIR" ]; then
  git checkout "$HOME"
  git branch -D "$BRANCH"
  git pull --ff-only origin "$HOME"
else
  WORKTREE_PATH=$(git rev-parse --show-toplevel)
  MAIN_PATH=$(git worktree list --porcelain | sed -n 's/^worktree //p' | head -n 1)
  if [ -z "$MAIN_PATH" ] || [ ! -d "$MAIN_PATH" ]; then echo "エラー: 主 worktree のパスを解決できません ($MAIN_PATH)。"; exit 1; fi
  cd "$MAIN_PATH" || { echo "エラー: 主 worktree への移動に失敗しました ($MAIN_PATH)。"; exit 1; }
  git worktree remove "$WORKTREE_PATH"
  git branch -D "$BRANCH"
  git fetch origin
fi
```

The return target (`HOME`) is resolved from the merged PR's `baseRefName` via `gh pr view`, so cleanup returns to whichever branch the PR targeted (`main`, `develop`, or a `release/*` branch). If the PR cannot be found, it falls back to `main`. `main`, `develop`, and `release/*` are protected: when the current branch is any of them, the skill reports "対象なし" and deletes nothing. Deleting a merged `release/*` branch is the responsibility of the release flow, not this skill.

## Steps (Reference)

### Step 1: Check Prerequisites

1. Check for uncommitted changes in the current worktree with `git status --porcelain`
   - If changes exist → display error and exit
2. Record the working branch with `git branch --show-current`
3. If the current branch is `main`, `develop`, or matches `release/*` → display `対象なし: <branch> は保護ブランチのため削除対象がありません。` and exit. All of them are protected and never deleted; deleting a merged `release/*` branch is the release flow's responsibility.
4. Resolve the return target (`HOME`) from the merged PR's base with `gh pr view <working branch> --json baseRefName -q .baseRefName`. If the PR cannot be found, fall back to `main`.
5. Detect whether the current worktree is the **primary worktree** (main clone) or a **linked worktree** by comparing `git rev-parse --git-dir` and `git rev-parse --git-common-dir` (equal → primary, differ → linked)

### Step 2A: Primary Worktree

If the current worktree is the primary worktree (the main clone):

1. `git checkout <home>`, then `git branch -D <working branch>`, then `git pull --ff-only origin <home>` to fast-forward the local `<home>` (the PR's base, `main` or `develop`) to `origin/<home>`. If the local `<home>` has unexpected commits ahead of `origin/<home>`, `--ff-only` causes the pull to fail so the user notices.

### Step 2B: Linked Worktree

If the current worktree is a linked worktree:

1. Capture the worktree path with `git rev-parse --show-toplevel`.
2. Capture the primary worktree path from the first `worktree` line of `git worktree list --porcelain`.
3. Validate that the primary worktree path is non-empty and exists, then `cd` into it. If the path is invalid or the `cd` fails, abort immediately — never proceed to `git worktree remove` while the CWD is still inside the worktree being removed, or it will delete the current worktree from the inside and leave the branch undeleted.
4. `git worktree remove <linked worktree path>` to remove the worktree directory.
5. `git branch -D <working branch>` to delete the branch.
6. Other linked worktrees and their branches are left alone.

### Step 3: Refresh Remote Refs

- **Primary worktree branch**: the local `<home>` working copy is already fast-forwarded to `origin/<home>` by Step 2A's `git pull --ff-only origin <home>`.
- **Linked worktree branch**: run `git fetch origin` to update remote-tracking refs only. The primary worktree's checked-out branch (which may or may not be `<home>`) is **not** touched — the user updates it when they next switch to that worktree.

### Step 4: Display Results

Display results in the following format:

```text
## ブランチクリーンアップ完了

- 削除した worktree: <path もしくは "なし（主 worktree）">
- 削除したブランチ: <branch 名>
- git fetch origin: <結果サマリ>
```

If the skill exited at Step 1 as "対象なし" (current branch is `main`/`develop`), report that instead and skip the deletion lines.
