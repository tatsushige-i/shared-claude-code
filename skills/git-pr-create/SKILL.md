---
name: git-pr-create
description: Create a GitHub PR from the current branch - analyze changes, check size limits, and generate PR with proper formatting
---

# Create PR Skill

現在のブランチからGitHub PRを作成するワークフローを自動化する。対応Issue特定・PR規模チェック・差分分析・PR作成を一連の流れで行う。

## 処理手順

### Step 1: 前提条件の確認

1. `git branch --show-current` で現在のブランチを取得する
   - `main` の場合 → 「エラー: mainブランチ上ではPRを作成できません。作業ブランチに切り替えてください。」と表示して終了
2. `gh pr list --head <ブランチ名> --json number,url,state` で既存PRを確認する
   - `OPEN` 状態のPRが存在する場合 → 「エラー: このブランチには既にオープンなPRがあります: <URL>」と表示して終了
3. `git status --porcelain` で未コミット変更を確認する
   - 変更がある場合 → 変更内容を分析し、関連ファイルを個別に `git add <ファイルパス>` でステージし、適切なコミットメッセージを生成して `git commit` する（`git add -A` や `git add .` は使わない。未追跡ファイルは変更内容との関連性を判断し、無関係なものは除外する）
4. `git log main..HEAD --oneline` でコミットの存在を確認する
   - コミットがない場合 → 「エラー: mainブランチからのコミットがありません。」と表示して終了
5. リモートにブランチをプッシュする:
   - `git push -u origin <ブランチ名>` を実行
   - 失敗した場合はエラーを表示して終了

### Step 2: 対応Issue特定

以下の順序でIssueを探索し、自動で特定する（ユーザーへの質問は行わない）:

1. `git log main..HEAD --format=%s%n%b` からコミットメッセージを取得し、`#(\d+)` パターンでIssue番号を探す
2. ブランチ名から探索する — ブランチ名に含まれる番号やキーワードで `gh issue list --search "<キーワード>" --json number,title,state` を使いIssueを検索する
3. `gh issue list --state open --limit 10 --json number,title,labels` で最近のオープンIssue一覧を取得し、ブランチ名・変更内容との関連性からIssueを推定する
4. 結果に応じて分岐:
   - **特定できた場合**: `gh issue view <番号> --json number,title,state` で検証し、タイトルを表示する。検証失敗（Issueが存在しない・クローズ済み）の場合はIssue無しとして続行する
   - **複数候補がある場合**: 以下の優先順位で自動選択する — (1) Issue番号がコミットメッセージに直接含まれるもの (2) ブランチ名のキーワードとタイトルが一致するもの (3) 最も新しいIssue
   - **特定できなかった場合**: Issue無しとして続行する（質問しない）

### Step 3: PR規模チェック

1. `git diff --numstat main...HEAD` で変更ファイル数・追加行数・削除行数を計測する
2. 自動生成ファイル（UIライブラリの生成ファイル等）は行数カウントから除外する
3. 共通開発規約の上限（10ファイル/300行）と比較する:
   - 超過している場合 → 警告とともにタスク分割の提案を表示して続行する（確認は求めない）
   - 新規ファイル・ディレクトリの作成が中心の場合はその旨を注記し、例外規定に該当する旨を明示する

### Step 4: 差分分析

1. `git log main..HEAD --oneline` と `git diff main...HEAD --stat` で変更内容を把握する
2. 必要に応じて `git diff main...HEAD` で詳細な差分を確認する
3. ブランチプレフィックスからPRタイプを推定する:

   | プレフィックス | PRタイトルのプレフィックス |
   |----------------|---------------------------|
   | `bugfix/`      | `fix: `                   |
   | `feature/`     | `feat: `                  |
   | `enhance/`     | `enhance: `               |
   | `docs/`        | `docs: `                  |
   | `chore/`       | `chore: `                 |

### Step 5: PR作成

1. PRタイトルを生成する（70文字以内、Step 4で推定したプレフィックスを使用）
2. PR本文を以下のテンプレートで生成する:

   ```
   ## Summary
   - <変更の要点1>
   - <変更の要点2>

   Closes #XX  ← Issue特定時のみ

   ## Test plan
   - [ ] <テスト項目>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

3. `gh pr create --base main --title "..." --body "..."` でPRを作成する
   - bodyはheredocを使用してフォーマットを保持する:
     ```
     gh pr create --base main --title "<タイトル>" --body "$(cat <<'EOF'
     <本文>
     EOF
     )"
     ```
4. 失敗した場合はエラーメッセージを表示して終了する

### Step 6: 結果表示

作成結果を以下の形式で表示する:

```
## PR作成完了

PR #XX: <タイトル>
<PR URL>

- 対応Issue: #XX <Issueタイトル>  ← Issue特定時のみ
- 変更ファイル数: X件
- 変更行数: +XX / -XX
```
