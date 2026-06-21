---
name: git-pr-finalize
description: Finalize a GitHub PR end-to-end - watch CI and Copilot review, address findings, merge after confirmation, and clean up branches
argument-hint: "[PR number]"
---

# PR Finalize Skill

PR 作成後のワークフローを一気通貫でオーケストレーションする: CI チェックと Copilot レビューを監視し、指摘に対応し、ユーザー確認のうえマージし、ブランチを整理する。

このスキルは**オーケストレーター**である。コメント対応やブランチ整理を再実装せず、既存スキルを再利用する:

- **`git-review-respond`** — レビューコメントの分析・コード修正・コミット・返信
- **`git-branch-cleanup`** — worktree スコープのローカル整理（`main`/`develop` は保護）

本スキルは **監視 + 完了判定 + マージ** に徹する。

## 処理手順

### Step 1: PR 番号の特定

- `$ARGUMENTS` が数値で指定されていれば、その PR 番号を使用する
- `$ARGUMENTS` が空（引数なし）の場合、現在ブランチから PR を推定する:
  - `gh pr view --json number,state,headRefName,title,url` を実行する（現在ブランチに紐づく PR を解決）
  - PR が見つかった場合、その旨を表示してその PR 番号を使用する:

    ```text
    Using PR #XX associated with the current branch `<branch>`.
    ```

  - PR が見つからない場合（コマンド失敗、または現在ブランチに PR が無い）、ユーザーに PR 番号を尋ねる:

    ```text
    Could not detect a PR from the current branch. Please provide the PR number you'd like to finalize.
    ```

- `$ARGUMENTS` が指定されているが数値でない場合、ユーザーに PR 番号を尋ねる:

  ```text
  Please provide the PR number you'd like to finalize.
  ```

- ユーザーの回答から明確な番号を特定できない場合、エラーを表示して終了する。推測や曖昧な解釈はしない:

  ```text
  Error: Could not identify the PR number. Please specify a number.
  ```

### Step 2: PR 情報の取得とブランチ切り替え

1. `gh pr view <PR number> --json state,headRefName,number,title,url,baseRefName,mergeable,mergeStateStatus` で PR 情報を取得する
2. コマンドが失敗した場合（指定番号の PR が存在しない）:
   - `gh issue view <PR number>` で Issue として存在するか確認し、Issue なら以下を表示して終了する:

     ```text
     Error: #XX is an Issue. Please specify a PR number.
     ```

   - Issue でもない場合、以下を表示して終了する:

     ```text
     Error: PR #XX does not exist.
     ```

3. PR の `state` を確認する:
   - `OPEN` でない場合 → 「This PR is already closed/merged.」と警告して終了する
4. PR の `headRefName` ブランチを checkout する:
   - `git checkout <headRefName>` を実行する
   - `git pull` でリモートの最新を取得する

### Step 3: 監視ループ（CI + Copilot レビュー）

CI チェックと Copilot レビューを同時に監視する。**両方**が落ち着く（CI 全パス かつ 未解決レビュースレッドが無い）まで、または停止条件に達するまでループを繰り返す。

1. **CI チェック**: `gh pr checks <PR number>` で PR のチェックをポーリングする（`gh pr checks <PR number> --watch` でチェック確定までブロックしてもよい）。
   - `pending` / `in_progress` は「実行中」として扱い、待機を続ける。
   - 実行中のチェックが無くなったら、結果を **全パス** か **失敗**（いずれかが failing/error 状態）に分類する。

2. **Copilot / 人間のレビュー**: GraphQL でレビュー状態を取得する。ベースは `git-review-respond` Step 3 と同じクエリで、スレッドを読む前に Copilot レビューの**完了**を確認できるよう `reviewRequests` と `reviews` を追加する:

   ```graphql
   query {
     repository(owner: "{owner}", name: "{repo}") {
       pullRequest(number: <PR number>) {
         reviewRequests(first: 10) {
           nodes {
             requestedReviewer {
               ... on Bot { login }
               ... on User { login }
             }
           }
         }
         reviews(first: 50) {
           nodes {
             state
             author { login }
             submittedAt
           }
         }
         reviewThreads(first: 100) {
           nodes {
             isResolved
             comments(first: 100) {
               nodes {
                 databaseId
               }
             }
           }
         }
       }
     }
   }
   ```

   - **まず Copilot レビューの完了をゲートする。** `reviewThreads` を評価する前に Copilot レビューの状態を分類する。`reviewThreads` が空でも「指摘なし」と結論づけてはならない — Copilot がまだ投稿していないだけの可能性がある:

     | 状態 | 検出条件 | アクション |
     |---|---|---|
     | Copilot レビュー進行中 | `reviewRequests` に `copilot-pull-request-reviewer` があるが `reviews.nodes` に Copilot のエントリがない | 待機を継続（ループ続行） |
     | Copilot レビュー完了 | `reviews.nodes` に Copilot のエントリがあり `submittedAt` が設定されている | 通常どおり `reviewThreads` を評価する |
     | Copilot 未設定 | `reviewRequests` にも `reviews` にも Copilot が存在しない | Copilot レビューなしで進めてよいかユーザーに確認する |

   - **ゲートを通過してからのみ**、未解決スレッドを検出する: 「未解決の指摘」 = `isResolved == false` の `reviewThread`。これには Copilot（`copilot-pull-request-reviewer`）と人間のレビューコメントの両方を含む。
   - 完了判定は**未解決スレッドの有無**で行う（再レビュー依頼の有無では判定しない）。

3. **ポーリング間隔 / タイムアウト**:
   - CI は通常数分、Copilot レビューも通常数分以内に到着する。およそ **30〜60 秒**間隔でポーリングする。
   - 監視フェーズごとに最大待機時間（例: 合計 **約 15 分**）を設ける。タイムアウト時は現在の CI とレビュー状態を表示して停止する — マージはしない。

4. ループ結果で分岐する:
   - **Copilot レビュー進行中**（ゲート未通過） → 完了またはタイムアウトまで待機を継続する（ループ続行）。
   - **CI 失敗** → Step 4（CI 失敗対応）へ。
   - **未解決レビュースレッドあり** → Step 4（レビュー指摘対応）へ。
   - **CI 全パス かつ Copilot レビューが確定（完了、または未設定でユーザー確認済み）かつ 未解決スレッド無し** → Step 5 へ。

### Step 4: 指摘対応（条件付き）

#### 4A: レビュー指摘あり

- この PR に対して `git-review-respond` スキルを実行する（`/git-review-respond <PR number>`）。分析・コード修正・コミット・push・各コメントへの返信を、同スキル自身のユーザー確認ステップに従って処理する。
- 完了後、**Step 3** に戻り再監視する（新規コミットで CI が再実行され、返信・解決でスレッド状態が更新される）。

#### 4B: CI 失敗

- 失敗ジョブのログを確認する（例: `gh run view <run id> --log-failed`、または `gh pr checks` のチェック詳細 URL をたどる）。
- 失敗原因と修正方針をユーザーに提示する。
- 最小限の修正を適用し、関連ファイルを個別に `git add <file path>` でステージし（`git add .` は使わない）、コミットして `git push` する。
- push 後、**Step 3** に戻り再監視する。

#### 停止条件

無限ループせず、以下の場合は停止して報告する:

- CI 失敗が自動で解消できない。
- PR がマージ不可状態（`mergeable == CONFLICTING`、マージコンフリクト、またはブロッキングな `mergeStateStatus`）。
- Step 3 の監視タイムアウトに達した。

現在の状態を明確に表示し、次の対応はユーザーに委ねる。

### Step 5: 完了判定

**CI 全パス** かつ **未解決レビュースレッドが無い** ことを確認する。これを満たした場合のみマージへ進む。

### Step 6: マージ（ユーザー確認必須）

マージは破壊的かつ外部に影響する操作である。**マージ前に必ずユーザーへ確認する**（共有規約「PR ステータス変更は明示指示が必要」と整合）。

1. マージ前サマリを提示し、明示的な承認を待つ:

   ```text
   ## Ready to Merge

   PR #XX: <title>
   <PR URL>

   - CI: all passing
   - Review threads: all resolved
   - Base: <baseRefName>

   Merge this PR? (the head branch will be deleted)
   ```

2. 承認後、`gh pr merge <PR number> --delete-branch` でマージする。
   - マージ方式はリポジトリ設定に従う（設定された squash/merge/rebase など）。ユーザーが指定しない限り方式を強制しない。
   - `--delete-branch` でリモートの head ブランチを削除する。base（`main`/`develop`）は決して削除しない。
3. マージが失敗した場合（マージ不可、ブランチ保護など）、エラーを表示して停止する。

### Step 7: ローカル整理

`git-branch-cleanup` スキル（`/git-branch-cleanup`）を実行し、ローカルの worktree と作業ブランチを整理する。

- 主クローンと linked worktree の双方を扱い、`main`/`develop` を保護する。
- マージ済み PR の `baseRefName` から戻り先を解決するため、PR が対象としたブランチへ戻る。

### Step 8: 完了サマリ

完了結果を以下の形式で表示する:

```text
## PR Finalized

PR #XX: <title>
<PR URL>

- Merge: <merged | not merged (reason)>
- Findings addressed: <summary, e.g., "2 review rounds, 1 CI fix" or "none">
- Remote branch: <deleted | n/a>
- Local cleanup: <summary from git-branch-cleanup>
```
