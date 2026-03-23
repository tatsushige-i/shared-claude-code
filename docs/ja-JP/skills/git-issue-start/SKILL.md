---
name: git-issue-start
description: Start implementation workflow from a GitHub Issue - fetch, validate labels, create branch, and enter plan mode
argument-hint: "<Issue number>"
---

# Implement Issue Skill

GitHub Issueの情報取得・ラベル検証・ブランチ作成・Plan Mode移行までのワークフローを自動化する。

## 処理手順

### Step 1: Issue番号の特定

- `$ARGUMENTS` が数値として指定されていればそのIssue番号を使用する
- 未指定または数値でない場合、ユーザーにIssue番号を尋ねる:
  ```
  対応するIssueの番号を教えてください。
  ```
- ユーザーの回答からも明確な数値を判別できない場合はエラーとして終了する。推測や曖昧な解釈は行わない:
  ```
  エラー: Issue番号を特定できませんでした。数値で指定してください。
  ```

### Step 2: Issue情報取得

1. `gh issue view <Issue番号> --json number,title,body,labels,state` でIssue情報を取得
2. コマンドが失敗した場合（番号に対応するIssueが存在しない場合）:
   - `gh pr view <Issue番号>` でPRとして存在するか確認し、PRの場合は以下を表示して終了:
     ```
     エラー: #XX はPRです。Issueの番号を指定してください。
     ```
   - PRでもない場合は以下を表示して終了:
     ```
     エラー: Issue #XX は存在しません。
     ```
3. `state` が `OPEN` でない場合 → 「このIssueは既にクローズされています」と警告して終了
4. 取得した情報を以下の形式で表示する:
   ```
   ## Issue #XX: <タイトル>

   ラベル: <ラベル一覧>

   <本文>
   ```

### Step 3: ラベル検証

共通開発規約（`conventions.md`）の「Issueラベルの必須化」セクションに基づき、以下を検証する:

1. **種類ラベル**: `bug`, `feature`, `enhancement`, `documentation`, `chore` のいずれか1つが付与されているか確認
2. **優先度ラベル**: `priority: high`, `priority: medium`, `priority: low` のいずれか1つが付与されているか確認
3. いずれかが不足している場合:
   - 不足しているラベルの種類をユーザーに伝え、どのラベルを付与するか確認する
   - ユーザーの回答に基づき `gh issue edit <Issue番号> --add-label "<ラベル>"` で付与する
4. 両方揃っている場合はそのまま次のステップに進む

### Step 4: ブランチ作成・チェックアウト

1. 共通開発規約（`conventions.md`）の「ブランチ命名規則」に基づき、種類ラベルからプレフィックスを決定する:

   | ラベル          | プレフィックス |
   |-----------------|----------------|
   | `bug`           | `bugfix/`      |
   | `feature`       | `feature/`     |
   | `enhancement`   | `enhance/`     |
   | `documentation` | `docs/`        |
   | `chore`         | `chore/`       |

2. Issueタイトルからブランチ名を生成する:
   - 日本語タイトルの場合は英語に変換する
   - kebab-case（ハイフン区切りの小文字英語）にする
   - `<プレフィックス><簡潔な説明>` の形式にする
3. 生成したブランチ名でそのままブランチを作成する:
   - `main` ブランチに切り替え: `git checkout main`
   - 最新を取得: `git pull`
   - ブランチ作成・チェックアウト: `git checkout -b <ブランチ名>`

### Step 5: プロジェクト固有のスキャフォールド（条件付き）

プロジェクトの `.claude/rules/architecture.md` にスキャフォールド手順（新規機能追加時のファイル生成パターン等）が定義されている場合は、その手順に従ってファイルを生成する。

定義がない場合、またはIssueの種類ラベルがスキャフォールド対象でない場合は、このステップをスキップする。

### Step 6: Plan Mode移行

1. `EnterPlanMode` ツールを呼び出してPlan Modeに移行する
2. 以下のメッセージを表示して実装計画の策定を促す:
   ```
   Plan Modeに移行しました。Issue #XX の実装計画を策定します。

   ## Issue情報
   - タイトル: <タイトル>
   - ラベル: <ラベル一覧>

   <Issue本文>

   上記のIssue内容に基づいて実装計画を策定します。

   **実装完了後の注意**: 実装が完了したら `git diff --stat` および主要な差分をユーザーに提示し、問題がなければ `/git-pr-create` でPRを作成するよう案内すること。コミットは `/git-pr-create` のフローに委ねる。
   ```
