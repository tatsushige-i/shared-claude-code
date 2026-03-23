---
name: config-claude-sync
description: Sync shared rules and skills from shared-claude-code repository - detect missing symlinks and create them
---

# Shared Config Sync Skill

shared-claude-codeリポジトリの共通ルール・スキルを現在のプロジェクトに同期する。未取り込みのシンボリックリンクを検出し、ユーザー確認の上で作成する。

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
4. 解決したパスに `rules/` と `skills/` ディレクトリが存在することを検証する
   - 存在しない場合 → エラーを表示して終了する

### Step 2: 差分の検出

1. shared-claude-codeの `rules/` 配下の `.md` ファイル一覧を取得する
2. shared-claude-codeの `skills/` 配下で `SKILL.md` を含むディレクトリ一覧を取得する
3. 現在のプロジェクトと比較し、未取り込みのものを検出する:
   - **ルール**: `.claude/rules/shared/<name>.md` にシンボリックリンクが存在しないもの
   - **スキル**: `.claude/skills/<name>` にシンボリックリンクが存在しないもの

### Step 3: 差分の提示・ユーザー確認

1. すべて同期済みの場合 → 以下を表示して終了する:
   ```
   すべてのルール・スキルは同期済みです。
   ```
2. 未取り込みがある場合、以下の形式で表示する:
   ```
   ## 未同期の項目

   ### ルール
   - conventions.md
   - security.md

   ### スキル
   - git-branch-cleanup
   - git-issue-create

   上記の項目を同期しますか？除外したい項目があればお知らせください。
   ```
3. ユーザーの回答に応じて分岐する:
   - 全承認 → 全項目を同期する
   - 一部除外を指示 → 指示されたものを除外して同期する

### Step 4: ブランチ作成

1. `git branch --show-current` で現在のブランチを確認する
2. **main以外の場合** → このステップをスキップしてStep 5へ進む（既存ブランチ上で作業中と判断）
3. **mainの場合**:
   - `git status --porcelain` で未コミット変更を確認する
     - 変更がある場合 → エラーを表示して終了する:
       ```
       エラー: mainブランチに未コミットの変更があります。変更をコミットまたはスタッシュしてから再実行してください。
       ```
   - `git branch --list chore/sync-claude-rules` で同名ブランチの存在を確認する
     - 存在しない場合 → `git checkout -b chore/sync-claude-rules` でブランチを作成する
     - 存在する場合 → `git checkout -b chore/sync-claude-rules-YYYYMMDD`（当日日付）でブランチを作成する

### Step 5: シンボリックリンク作成

1. ルールの同期:
   - `.claude/rules/shared/` ディレクトリが存在しない場合は `mkdir -p` で作成する
   - 既存のルールシンボリックリンクの `readlink` 結果からプレフィックスを取得し、同じパターンで新しいシンボリックリンクを作成する
   - 例: 既存リンクが `../../../shared-claude-code/rules/conventions.md` なら、新規も `../../../shared-claude-code/rules/<name>.md` で作成
2. スキルの同期:
   - 既存のスキルシンボリックリンクの `readlink` 結果からプレフィックスを取得し、同じパターンで新しいシンボリックリンクを作成する
   - 例: 既存リンクが `../../shared-claude-code/skills/git-pr-create` なら、新規も `../../shared-claude-code/skills/<name>` で作成
3. 各シンボリックリンク作成後、リンク先が正しく解決できることを確認する

### Step 6: ドキュメントの更新

Step 5で新規スキルが同期された場合のみ実行する。ルールのみの同期時はスキップする。

1. プロジェクト内に `.claude/skills/README.md` が存在するか確認する
   - 存在する場合:
     - 同期した各スキルについて、スキル名がファイル内に既に記載されているか確認する
     - 未記載のスキルについて、shared-claude-codeの `skills/<name>/SKILL.md` フロントマターから `description` を取得する
     - 既存テーブルの末尾に `| \`<name>\` | \`/<name>\` | <description> |` の行を追記する
     - `git add .claude/skills/README.md` でステージする
   - 存在しない場合: スキップ
2. プロジェクト内に `docs/ja-JP/skills/README.md` が存在するか確認する
   - 存在する場合: 同様に未記載スキルを検出し、テーブル末尾に追記してステージする
   - 存在しない場合: スキップ
3. プロジェクトルートの `CLAUDE.md` にスキル一覧（テーブルやリスト等）が記載されているか確認する
   - 記載されている場合:
     - 未記載の同期スキルについて、既存のフォーマットに合わせてエントリを追記する
     - `git add CLAUDE.md` でステージする
   - `CLAUDE.md` が存在しない場合、またはスキル一覧が含まれていない場合: スキップ

### Step 6b: 共通Hooksの同期

1. shared-claude-codeリポジトリの `hooks/shared-hooks.json` を読み取る
   - ファイルが存在しない場合 → このステップをスキップする
2. プロジェクトの `.claude/settings.json` を読み取る
   - 存在しない場合 → `{}` として扱う
3. `shared-hooks.json` の各hookエントリについて重複チェックを行う:
   - ファイル構造は `settings.json` の `hooks` セクションと同じ形式で、各hookエントリに `_id`・`_description`・`_detect_by` のメタデータフィールドが付与されている
   - `matcher` が設定されているエントリ: 同一イベントキー + `matcher` の既存hookを検索する
   - `matcher` がないエントリ（Stopなど）: 同一イベントキーの既存hookすべてを対象にする
   - 既存hookの `command` 文字列が `_detect_by` の正規表現パターンにマッチする場合 → 「既存」とみなす
   - マッチしない場合 → 「追加対象」としてリストアップする
4. 追加対象があれば、ユーザーに提示して確認する:
   ```
   ## 未同期のHooks
   - block-force-push (PreToolUse/Bash): Block git force push
   - notify-stop (Stop): macOS notification on task completion

   上記のhooksを同期しますか？除外したい項目があればお知らせください。
   ```
   - すべて既存の場合 → `すべてのhooksは同期済みです。` と表示してStep 7へ進む
5. 承認されたhooksを `.claude/settings.json` に追加する:
   - 挿入前に `_id`・`_description`・`_detect_by` フィールドを除去する
   - event/matcher 単位で既存の `hooks` オブジェクトにマージする（既存エントリは上書きしない）
   - `.claude/settings.json` が存在しない場合 → hooksのみの構造で新規作成する
   - `hooks` キーが存在しない場合 → 追加する
6. ファイルをステージする: `git add .claude/settings.json`

### Step 7: コミット

1. Step 5で作成したシンボリックリンクを個別にステージする（`git add -A` や `git add .` は使わない）:
   - ルール: `git add .claude/rules/shared/<name>.md`
   - スキル: `git add .claude/skills/<name>`
   - Step 6でステージしたREADMEファイルはそのまま含まれる
   - Step 6bでステージした `settings.json` はそのまま含まれる
2. `git commit -m "chore: sync shared claude rules and skills"` でコミットする
3. コミットが失敗した場合（ステージされたファイルがない等）は警告を表示してStep 8へ進む

### Step 8: 結果表示

同期結果を以下の形式で表示する:

```
## 同期完了

- ブランチ: <ブランチ名>（新規作成 / 既存）
- コミット: <コミットハッシュ短縮>
- 同期したルール: X件
  - <ファイル名1>
  - <ファイル名2>
- 同期したスキル: X件
  - <スキル名1>
  - <スキル名2>
- 同期したhooks: X件
  - <hook ID1>
  - <hook ID2>
- 更新したREADME: <更新ファイル一覧>

`/git-pr-create` でPRを作成できます。
```

- Step 4でブランチを新規作成した場合のみ「`/git-pr-create` でPRを作成できます。」を表示する
- main以外のブランチで実行した場合は「既存」と表示し、PR案内は省略する
- Step 6でREADMEを更新した場合のみ「更新したREADME」を表示する
- Step 6bでhooksを追加した場合のみ「同期したhooks」を表示する
