---
name: docs-validate
description: Validate documentation consistency across the repository - check skills, rules, symlinks, README structure, and translations
---

# ドキュメント整合性チェックスキル

shared-claude-code リポジトリのドキュメント整合性を検証する。スキル、ルール、シンボリックリンク、README の Structure セクション、日本語翻訳をスキャンし、不整合を検出する。可能な場合は自動修正を提案する。

## 処理手順

### Step 1: スキルのスキャン

1. `skills/` 配下のすべてのディレクトリを列挙する（`README.md` を除く）
2. 各スキルディレクトリに対して以下を検証する:
   - **1-1 SKILL.md の存在とフロントマター**: `skills/<name>/SKILL.md` が存在し、YAML フロントマターに `name` と `description` が含まれている
   - **1-2 symlink の存在**: `.claude/skills/<name>` がシンボリックリンクとして存在し、`../../skills/<name>` を指している
     - `readlink .claude/skills/<name>` で確認する
     - シンボリックリンクが存在しないか、異なるターゲットを指している場合 → 問題としてフラグを立てる
   - **1-3 README テーブルのエントリ**: `skills/README.md` のスキルテーブルにスキル名の行が含まれている
     - `grep` でスキル名を `skills/README.md` 内で検索する
   - **1-4 日本語翻訳**: `docs/ja-JP/skills/<name>/SKILL.md` が存在する
3. 検出された各問題をカテゴリと詳細と共に記録する

### Step 2: ルールのスキャン

1. `rules/` 配下のすべての `.md` ファイルを列挙する（`README.md` がある場合は除く）
2. 各ルールファイルに対して以下を検証する:
   - **2-1 日本語翻訳**: `docs/ja-JP/rules/<filename>` が存在する
3. 検出された各問題を記録する

### Step 3: Structure セクションの検証

1. **README.md のスキル一覧**:
   - `README.md` の Structure セクションをパースし、記載されているスキル名を抽出する
   - `skills/` 配下の実際のディレクトリと比較する
   - ディスク上に存在するが Structure セクションに記載されていないスキルをフラグする
   - Structure セクションに記載されているがディスク上に存在しないエントリをフラグする
2. **日本語版 README.md のスキル一覧**:
   - `docs/ja-JP/README.md` の Structure セクションをパースする
   - 英語版 `README.md` の Structure セクションと比較する
   - 2つのリスト間の差異をフラグする

### Step 4: 主要翻訳ファイルの確認

以下の主要翻訳ファイルが存在することを確認する:

| # | ファイル | チェック内容 |
|---|---|---|
| 4-1 | `docs/ja-JP/README.md` | ファイルが存在する |
| 4-2 | `docs/ja-JP/CLAUDE.md` | ファイルが存在する |
| 4-3 | `docs/ja-JP/skills/README.md` | ファイルが存在する |

### Step 5: 結果の提示

1. すべてのカテゴリで問題が見つからなかった場合 → 以下を表示する:
   ```
   すべてのドキュメント整合性チェックに合格しました。
   ```
2. 問題が見つかった場合 → カテゴリ別に結果を表示する:
   ```
   ## ドキュメント整合性チェック結果

   ### Skills 整合性
   - `config-github-sync` — symlink 欠落
   - `git-pr-create` — 日本語翻訳なし

   ### Rules 整合性
   - すべて合格 ✓

   ### Structure セクション
   - README.md — `config-github-sync` が Structure に未記載

   ### 翻訳ファイル
   - すべて合格 ✓

   自動修正可能な項目があります。修正しますか？（全て / 選択 / スキップ）
   ```
3. 次のステップに進む前にユーザーの応答を待つ

### Step 6: 修正の適用

ユーザーの応答に基づいて修正を実施する:

**自動修正可能**（ユーザーの承認後に適用）:
- **symlink 欠落**: `ln -s ../../skills/<name> .claude/skills/<name>` で作成し、`readlink` で検証する
- **skills/README.md のエントリ欠落**: `skills/<name>/SKILL.md` のフロントマターから `name` と `description` を読み取り、`| \`<name>\` | \`/<name>\` | <description> |` 形式のテーブル行を生成して `skills/README.md` のテーブルに追記する

**手動修正が必要**（警告のみ表示）:
- **Structure セクションの不整合**: 更新が必要なファイルと、欠落/余分なスキルを表示する
- **日本語翻訳の欠落**: 作成が必要なファイルを表示する

修正適用後に結果を表示する:

```
## 修正完了

- symlink 作成: .claude/skills/config-github-sync
- README テーブル行追加: config-github-sync

手動対応が必要:
- README.md の Structure セクションに `config-github-sync` を追加してください
- 日本語翻訳を作成してください: docs/ja-JP/skills/config-github-sync/SKILL.md
```
