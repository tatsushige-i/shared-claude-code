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

  ```text
  対応するIssueの番号を教えてください。
  ```

- ユーザーの回答からも明確な数値を判別できない場合はエラーとして終了する。推測や曖昧な解釈は行わない:

  ```text
  エラー: Issue番号を特定できませんでした。数値で指定してください。
  ```

### Step 2: Issue情報取得

1. `gh issue view <Issue番号> --json number,title,body,labels,state` でIssue情報を取得
2. コマンドが失敗した場合（番号に対応するIssueが存在しない場合）:
   - `gh pr view <Issue番号>` でPRとして存在するか確認し、PRの場合は以下を表示して終了:

     ```text
     エラー: #XX はPRです。Issueの番号を指定してください。
     ```

   - PRでもない場合は以下を表示して終了:

     ```text
     エラー: Issue #XX は存在しません。
     ```

3. `state` が `OPEN` でない場合 → 「このIssueは既にクローズされています」と警告して終了
4. 取得した情報を以下の形式で表示する:

   ```text
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

### Step 4: base ブランチとプレフィックスの決定

このスキルは base ブランチを自動判定し、`main` のみのリポジトリ（後方互換）でも、`main` + `develop`（git-flow ライト）のリポジトリでも、`release/*` ブランチを持つリポジトリ（標準 Git Flow）でも動作する。

1. `origin` で利用可能な統合ブランチを検出する:
   - `develop`: `git ls-remote --heads origin develop` を実行（出力が非空 → あり）
   - `release/*`: `git ls-remote --heads origin 'release/*'` を実行（出力が非空 → 1 つ以上あり）

2. base ブランチとプレフィックスを決定する（最初に一致したルールを採用）:

   | # | 条件                                                         | base ブランチ        | プレフィックス             |
   |---|--------------------------------------------------------------|----------------------|----------------------------|
   | 1 | Issue に `hotfix` ラベル **かつ** `develop` が存在            | `main`               | `hotfix/`                  |
   | 2 | Issue に `release` ラベル **かつ** `release/*` が 1 つ以上存在 | 最新の `release/*`   | 下の種類ラベル表に従う     |
   | 3 | `develop` が存在                                             | `develop`            | 下の種類ラベル表に従う     |
   | 4 | それ以外                                                     | `main`               | 下の種類ラベル表に従う     |

   - 最新の `release/*`（ルール 2）はバージョン番号が最大のもの: `git ls-remote --heads origin 'release/*' | sed -n 's#.*refs/heads/##p' | sort -V | tail -n 1`。
   - 後方互換: `develop` も `release/*` も持たないリポジトリは常にルール 4（base は `main`）に落ちるため、従来どおりの挙動。`release` ラベルは `release/*` ブランチが存在しない限り効果がなく、`hotfix` ラベルは `develop` が存在しない限り効果がない。

3. 種類ラベルのプレフィックス表（`hotfix/` が適用されない場合に使用）。共通開発規約（`conventions.md`）の「ブランチ命名規則」に基づく:

   | ラベル          | プレフィックス |
   |-----------------|----------------|
   | `bug`           | `bugfix/`      |
   | `feature`       | `feature/`     |
   | `enhancement`   | `enhance/`     |
   | `documentation` | `docs/`        |
   | `chore`         | `chore/`       |

4. Issueタイトルからブランチ名を生成する:
   - 日本語タイトルの場合は英語に変換する
   - kebab-case（ハイフン区切りの小文字英語）にする
   - `<プレフィックス><簡潔な説明>` の形式にする
5. 生成したブランチ名でそのままブランチを作成する:
   - base ブランチに切り替え: `git checkout <base>`
   - 最新を取得: `git pull --ff-only`
   - ブランチ作成・チェックアウト: `git checkout -b <ブランチ名>`

### Step 5: プロジェクト固有のスキャフォールド（条件付き）

プロジェクトの `.claude/rules/architecture.md` にスキャフォールド手順（新規機能追加時のファイル生成パターン等）が定義されている場合は、その手順に従ってファイルを生成する。

定義がない場合、またはIssueの種類ラベルがスキャフォールド対象でない場合は、このステップをスキップする。

### Step 6: Plan Mode移行

1. `EnterPlanMode` ツールを呼び出してPlan Modeに移行する
2. 以下のメッセージを表示して実装計画の策定を促す:

   ```text
   Plan Modeに移行しました。Issue #XX の実装計画を策定します。

   ## Issue情報
   - タイトル: <タイトル>
   - ラベル: <ラベル一覧>

   <Issue本文>

   上記のIssue内容に基づいて実装計画を策定します。

   **実装フェーズの支援**: 計画承認後、`/impl-pipeline <Issue番号>` を使うと実装支援エージェント群（planner/coder/tester）とレビュー連携のパイプラインを利用できる（機械的なIssueは自律実装、判断・UI系は対話実装に切り分け、人間ゲートを維持）。

   **実装完了後の注意**: 実装が完了したら `git diff --stat` および主要な差分をユーザーに提示し、問題がなければ `/git-pr-create` でPRを作成するよう案内すること。コミットは `/git-pr-create` のフローに委ねる。
   ```
