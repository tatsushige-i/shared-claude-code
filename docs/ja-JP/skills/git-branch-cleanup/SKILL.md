---
name: git-branch-cleanup
description: Worktree-scoped cleanup — remove the current worktree and its working branch only, leaving other worktrees and branches untouched
---

# Branch Cleanup Skill

PR マージ後に **現在の worktree** と **そこで checkout している作業ブランチのみ** を片付けるスキル。

このスキルは、Claude Code と Codex がそれぞれ独自の clone / worktree で並行して作業し、複数の Issue が同時進行している運用を前提とする。他のローカルブランチや他の worktree（Claude Code のメインクローン、別 Issue の Codex worktree など）は **絶対に削除せず**、それらが存在しても本スキルは停止しない。

## 対象スコープ

- **対象**: このスキルを実行した worktree と、そこで現在 checkout されている作業ブランチ。
- **対象外**: 他のローカルブランチ、他の linked worktree、他の clone。検査も変更もしない。
- 引数は受け取らない。

## 実行方針

このスキルはユーザーが明示的に `/git-branch-cleanup` を実行した時点で全操作が承認済みである。以下を厳守すること:

- **全操作を 1 つの Bash tool call のコマンドチェーンで実行する。Bash tool call を分割してはならない**
- `gh pr view`, `git checkout <home>`, `git worktree remove`, `git branch -D`, `git fetch origin`, `git pull --ff-only origin <home>` はすべて事前承認済みの操作として扱う
- ステップ間で「実行してよいですか？」「続行しますか？」等の質問を挟まない
- エラーが発生した場合、または「対象なし（現在ブランチが保護対象の `main`/`develop`）」を検出した場合のみ停止する

## 実行コマンド

以下のコマンドチェーンを **1 つの Bash tool call** でそのまま実行する:

```bash
set -e
STATUS=$(git status --porcelain)
if [ -n "$STATUS" ]; then echo "エラー: 未コミットの変更があります。"; exit 1; fi
BRANCH=$(git branch --show-current)
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "develop" ]; then
  echo "対象なし: $BRANCH は保護ブランチのため削除対象がありません。"; exit 0
fi
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
  MAIN_PATH=$(git worktree list --porcelain | awk '/^worktree/{print $2; exit}')
  cd "$MAIN_PATH"
  git worktree remove "$WORKTREE_PATH"
  git branch -D "$BRANCH"
  git fetch origin
fi
```

戻り先（`HOME`）はマージ済み PR の `baseRefName` を `gh pr view` で取得して確定するため、PR が対象としたブランチ（`main` または `develop`）に戻る。PR が見つからない場合は `main` にフォールバックする。`main` と `develop` の両方を保護対象とし、現在ブランチがいずれかの場合は「対象なし」を表示して何も削除しない。

## 処理手順（参考）

### Step 1: 前提条件の確認

1. 現在の worktree で `git status --porcelain` を実行し、未コミット変更を確認する
   - 変更がある場合 → エラーを表示して終了する
2. `git branch --show-current` で作業ブランチ名を保持する
3. 現在ブランチが `main` または `develop` → `対象なし: <ブランチ> は保護ブランチのため削除対象がありません。` を表示して終了する。両方とも保護対象で、削除しない。
4. マージ済み PR の base から戻り先（`HOME`）を `gh pr view <作業ブランチ> --json baseRefName -q .baseRefName` で確定する。PR が見つからない場合は `main` にフォールバックする。
5. `git rev-parse --git-dir` と `git rev-parse --git-common-dir` を比較して、現在の worktree が **主 worktree（メインクローン）** か **linked worktree** かを判定する（一致 → 主、不一致 → linked）

### Step 2A: 主 worktree の場合

主 worktree（メインクローン）で実行された場合:

1. `git checkout <home>` の後、`git branch -D <作業ブランチ>` を実行し、続けて `git pull --ff-only origin <home>` でローカル `<home>`（PR の base、`main` または `develop`）を `origin/<home>` に fast-forward する。ローカル `<home>` に想定外のコミットがある場合は `--ff-only` により失敗して停止し、ユーザが気付ける。

### Step 2B: linked worktree の場合

linked worktree で実行された場合:

1. `git rev-parse --show-toplevel` で対象 worktree のパスを取得する。
2. `git worktree list --porcelain` の先頭 `worktree` 行から主 worktree のパスを取得する。
3. `cd` で主 worktree へ移動する。
4. `git worktree remove <linked worktree のパス>` で worktree ディレクトリを削除する。
5. `git branch -D <作業ブランチ>` でブランチを削除する。
6. 他の linked worktree やそのブランチは一切触らない。

### Step 3: リモート参照の更新

- **主 worktree 分岐**: Step 2A の `git pull --ff-only origin <home>` ですでにローカル `<home>` を `origin/<home>` に fast-forward 済み。
- **linked worktree 分岐**: `git fetch origin` で remote-tracking refs のみ更新する。主 worktree で checkout 中のブランチ（`<home>` とは限らない）には触れない。ユーザーが次にその worktree へ戻ったタイミングで pull する想定。

### Step 4: 結果表示

完了結果を以下の形式で表示する:

```text
## ブランチクリーンアップ完了

- 削除した worktree: <path もしくは "なし（主 worktree）">
- 削除したブランチ: <ブランチ名>
- git fetch origin: <結果サマリ>
```

Step 1 で「対象なし」終了した場合（現在ブランチが `main`/`develop`）は、その旨を表示し削除関連の行は省略する。
