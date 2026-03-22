---
name: config-github-sync
description: Sync .github files and ci-templates from shared-claude-code repository - detect language, present diffs, and copy files
---

# GitHub Config Sync Skill

shared-claude-codeリポジトリの `.github/` 配下の共有資材（ISSUE_TEMPLATE、workflows）を現在のプロジェクトにコピーベースで同期する。GitHubがシンボリックリンクを認識しないため、ファイルコピーで同期する。

## 処理手順

### Step 1: 共通リポジトリの特定

1. `.claude/rules/shared/` および `.claude/skills/` 配下のシンボリックリンクを探索する
2. 見つかったシンボリックリンクの `readlink` 結果からshared-claude-codeリポジトリのパスを解決する
   - ルールのリンク例: `../../../shared-claude-code/rules/conventions.md` → `shared-claude-code` のパスを抽出
   - スキルのリンク例: `../../shared-claude-code/skills/git-pr-create` → `shared-claude-code` のパスを抽出
3. シンボリックリンクが1つも見つからない場合 → エラーを表示して終了する:
   ```
   エラー: shared-claude-codeへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. 解決したパスに `.github/` ディレクトリが存在することを検証する
   - 存在しない場合 → エラーを表示して終了する

### Step 2: 差分の検出

1. shared側の `.github/ISSUE_TEMPLATE/` と `.github/workflows/` を走査する
2. ローカルの対応ファイルと比較し、各ファイルの状態を判定する:
   - **同一**: 内容が一致（`diff -q` で判定）
   - **差分あり**: 両方に存在するが内容が異なる
   - **新規**: shared側にあるがローカルに存在しない
   - **ローカルのみ**: ローカルにあるがshared側に存在しない（警告表示のみ、削除はしない）

### Step 3: 差分の提示・ユーザー確認

1. すべて同一の場合 → 以下を表示して終了する:
   ```
   .github/ 配下のファイルはすべて同期済みです。
   ```
2. 差分がある場合、カテゴリ別に状態を一覧表示する:
   ```
   ## .github 同期チェック

   ### ISSUE_TEMPLATE
   - bug.yml — 同一 ✓
   - chore.yml — ローカルに存在しない（新規追加）
   - enhancement.yml — 差分あり

   ### workflows
   - ci.yml — 差分あり

   同期する項目を選択してください（例: 全て / ISSUE_TEMPLATEのみ）。
   差分の詳細を確認したい場合はお知らせください。
   ```
3. ユーザーが差分詳細を要求した場合 → `diff` 出力を表示する
4. ユーザーの回答に応じて同期対象を決定する:
   - 全承認 → 全項目を同期する
   - 一部除外を指示 → 指示されたものを除外して同期する

### Step 4: ファイルコピー

1. `.github/ISSUE_TEMPLATE/` や `.github/workflows/` ディレクトリが存在しない場合は `mkdir -p` で作成する
2. 選択された項目を `cp` で上書きコピーする
3. コピー後、`diff -q` で内容が一致することを検証する
   - 不一致の場合 → エラーを表示する

### Step 5: CI テンプレート同期

1. shared リポジトリに `ci-templates/` ディレクトリが存在するか確認する。存在しない場合はこのステップをスキップする。
2. **プロジェクトの言語/フレームワークを推定する**（ローカルのプロジェクトルートを確認）:
   - `package.json` あり → Node.js 系。`dependencies.next` または `devDependencies.next` があれば **Next.js** と判定
   - `pyproject.toml` または `setup.py` あり → **Python**
   - `go.mod` あり → **Go**
   - `ci-templates/` 配下の利用可能なテンプレートディレクトリも候補として列挙する
3. **候補を提示してユーザーに確認する**:
   ```
   ## CI テンプレート同期

   推定フレームワーク: Next.js（package.json の dependencies.next を検出）

   利用可能なテンプレート:
   - nextjs（推奨）

   同期するテンプレートを選択してください（スキップする場合は「スキップ」と入力）。
   ```
   - 一致するテンプレートがない場合 → 利用可能なテンプレートを表示し、選択またはスキップを促す
4. **選択されたテンプレートの差分を検出する**:
   - `ci-templates/<lang>/.github/workflows/ci.yml` ↔ ローカルの `.github/workflows/ci.yml`
   - `ci-templates/<lang>/` 直下の各ファイル（`.github/` 以外）↔ ローカルのプロジェクトルートの同名ファイル
   - 各ファイルを **同一**・**差分あり**・**新規**・**ローカルのみ** に分類（Step 2 と同じ基準）
5. **差分を提示してユーザーに確認する**:
   - すべて同一の場合 → 「CI テンプレートのファイルはすべて同期済みです。」と表示してコピーをスキップ
   - 差分がある場合、カテゴリ別に状態を一覧表示する:
     ```
     ### CI テンプレート（nextjs）
     #### .github/workflows
     - ci.yml — 差分あり

     #### プロジェクトルート
     - package.json — ローカルに存在しない（新規追加）
     - tsconfig.json — 差分あり
     - jest.config.mjs — 同一 ✓

     同期する項目を選択してください（例: 全て / ci.ymlのみ）。
     差分の詳細を確認したい場合はお知らせください。
     ```
   - ユーザーが差分詳細を要求した場合 → `diff` 出力を表示する
   - ユーザーの回答に応じて同期対象を決定する
6. **ファイルをコピーする**:
   - コピー先ディレクトリが存在しない場合は `mkdir -p` で作成する
   - 選択されたファイルを `cp` で上書きコピーする
   - コピー後、`diff -q` で内容が一致することを検証する
     - 不一致の場合 → エラーを表示する

### Step 6: ラベル色の同期

標準ラベル色を以下のように定義する（出典: shared-claude-code リポジトリ）:

| ラベル名 | カラーコード |
|---|---|
| `bug` | `d73a4a` |
| `feature` | `0e8a16` |
| `enhancement` | `a2eeef` |
| `documentation` | `0075ca` |
| `chore` | `ededed` |
| `priority: high` | `b60205` |
| `priority: medium` | `fbca04` |
| `priority: low` | `0e8a16` |

1. `gh label list --json name,color` で対象リポジトリのラベル一覧を取得する
2. 上記の標準テーブルの各ラベルについて、対象リポジトリに存在するか確認する:
   - **対象リポジトリに存在しない**: スキップ（作成はしない）
   - **色が異なる**: 修正対象としてフラグを立てる
   - **色が一致**: `✓` としてマークする
3. 見つかったすべてのラベルが一致する場合 → 以下を表示してStep 7へ進む:
   ```
   ラベル色はすべて標準に一致しています。
   ```
4. 差分がある場合、表示してユーザーに確認する:
   ```
   ## ラベル色チェック

   - `bug`: d73a4a → e11d48 （差分あり）
   - `feature`: 0e8a16 ✓
   - `priority: high`: b60205 ✓

   ラベル色を修正しますか？（はい / スキップ）
   ```
5. ユーザーが承認した場合 → 差分のある各ラベルに対して `gh label edit <name> --color <hex>` を実行する
6. ユーザーがスキップした場合 → 変更せずにStep 7へ進む

### Step 7: 結果表示

同期結果を以下の形式で表示する:

```
## 同期完了

- 同期したISSUE_TEMPLATE: X件
  - <ファイル名1>（新規追加）
  - <ファイル名2>（更新）
- 同期したワークフロー: X件
  - <ファイル名1>（更新）
- 同期したCIテンプレート（<lang>）: X件
  - .github/workflows/ci.yml（更新）
  - tsconfig.json（新規追加）
- 修正したラベル色: X件
  - `bug`: e11d48 → d73a4a
```

CI テンプレートを同期しなかった場合は該当行を省略する。ラベル色の修正がなかった場合も該当行を省略する。
