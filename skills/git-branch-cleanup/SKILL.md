---
name: git-branch-cleanup
description: Clean up local branches - switch to main, delete all local branches, and pull latest
---

# Branch Cleanup Skill

PR作成・マージ後のローカルブランチクリーンアップを自動化する。mainへの切り替え・ローカルブランチ削除・最新取得を一連の流れで行う。

## 実行方針

このスキルはPRマージ後のクリーンアップ用途であり、**全ステップを確認なしで一連の流れとして実行する**。`git branch -D` 等の破壊的操作も含め、途中でユーザーに承認を求めない。

## 処理手順

### Step 1: 前提条件の確認

1. `git status --porcelain` で未コミット変更を確認する
   - 変更がある場合 → エラーを表示して終了する
2. `git branch --show-current` で現在のブランチを取得する
   - 既に `main` の場合 → ブランチ切り替え（Step 2）をスキップする

### Step 2: mainブランチへ切り替え

1. `git checkout main` を実行する
2. 失敗した場合はエラーを表示して終了する

### Step 3: ローカルブランチの削除

1. `git branch` でmain以外のローカルブランチ一覧を取得する
   - ブランチがない場合 → 「削除対象のブランチはありません」と表示してStep 4へ進む
2. 各ブランチを `git branch -D <ブランチ名>` で削除する

### Step 4: 最新取得

1. `git pull` を実行する
2. 失敗した場合はエラーを表示する

### Step 5: 結果表示

完了結果を以下の形式で表示する:

```
## ブランチクリーンアップ完了

- 現在のブランチ: main
- 削除したブランチ: X件
  - <ブランチ名1>
  - <ブランチ名2>
- git pull: <結果サマリ>
```
