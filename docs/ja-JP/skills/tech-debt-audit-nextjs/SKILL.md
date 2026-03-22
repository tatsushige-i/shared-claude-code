---
name: tech-debt-audit-nextjs
description: Next.js（App Router）プロジェクトの技術的負債を調査し、優先度付きレポートを生成する
---

# Next.js 技術的負債調査スキル

Next.js（App Router）プロジェクトの技術的負債を調査する。`rules/tech-debt-checklist.md` の各カテゴリをNext.js固有の手順で調査し、フレームワーク固有のチェックも実施する。ファイルパス・行番号・改善案を含む優先度付きレポートを出力する。

## Steps

### Step 1: プロジェクト検出

1. `next.config.*` の存在と `package.json` の `next` 依存を確認し、Next.jsプロジェクトであることを検証する
   - Next.jsプロジェクトでない場合 → 「エラー: Next.jsプロジェクトではありません。」と表示して終了
2. `app/` ディレクトリの存在でApp Routerの使用を確認する
   - `pages/` のみ存在する場合 → 「警告: このプロジェクトはPages Routerを使用しています。このスキルはApp Routerを対象としています。」と表示して終了
3. プロジェクト構造をスキャンしてレイアウトを把握する:
   - `app/` 配下のルートセグメントを一覧化
   - `src/` 使用 vs ルートレベル構成の識別
   - `lib/`、`components/`、`hooks/`、`utils/` 等のディレクトリの有無を確認

### Step 2: 共通チェックリスト調査

`rules/tech-debt-checklist.md` の各カテゴリをNext.jsコンテキストで調査する:

#### コードの重複
- コンポーネント間で繰り返されるJSXパターンの検索（類似のフォームレイアウト、カード構造等）
- ルートセグメント間で重複するデータ取得ロジックの確認
- `lib/`、`utils/`、`helpers/` ディレクトリ間で重複するユーティリティ関数の検索

#### アーキテクチャとレイヤー分離
- コンポーネント内での直接的なデータベースやORM呼び出しの確認（Server ActionsやAPIルートに配置すべき）
- `"use client"` コンポーネントがサーバー専用モジュールをインポートしていないか検証
- `madge` またはインポートチェーンの手動検査による循環インポートの検出

#### エラーハンドリング
- `error.tsx` 境界が欠如しているルートセグメントの一覧化
- APIルートハンドラ（`route.ts`）でのtry-catchやエラーレスポンスの欠如を確認
- 空のcatchブロックや握り潰されたエラーの検索

#### 型安全性
- `.ts` および `.tsx` ファイル全体での `any` 型使用の検索
- コンポーネントpropsの型定義欠如の確認（インライン `props: any` や型なしのデストラクチャリング）
- 適切な型付けを回避する型アサーション（`as`）の検出

#### デッドコード
- TypeScriptやESLintレポートが利用可能な場合、未使用エクスポートの特定
- コメントアウトされたコードブロックの検索
- 未使用インポートの確認

#### 定数と設定
- マジックナンバーやハードコードされた文字列リテラル（URL、APIエンドポイント、閾値）の検索
- `process.env` や `next.config.*` を使用していない環境固有の値の確認
- ファイル間で重複する定数定義の検出

#### コンポーネント / モジュールの肥大化
- 300行を超える `.tsx` ファイルの特定
- 100行を超える関数やコンポーネントのフラグ付け
- 深くネストされたJSX（4段階以上）の確認

#### 依存関係の管理
- `package.json` の未使用の可能性がある依存関係の確認
- 同じ機能を提供する重複パッケージ（例: 複数の日付ライブラリ）の検出
- 非推奨警告が出ている依存関係の確認

#### テスト
- ルート/コンポーネント数とテストファイル数の比較によるカバレッジギャップの推定
- テスト設定（`jest.config.*`、`vitest.config.*` 等）の存在確認
- テストが欠如しているクリティカルパス（認証、決済、データ変更）の特定

#### アクセシビリティ
- `alt` 属性が欠如した `<img>` タグの検索（`next/image` と `alt` を使用すべき）
- `aria-*` ラベルが欠如したインタラクティブ要素（`<button>`、`<a>`、`<input>`）の確認
- 非インタラクティブ要素へのクリックハンドラ（`<div onClick>`）の検出

#### パフォーマンス
- Server Componentで十分なコンポーネントへの不要な `"use client"` ディレクティブの確認
- `next/image` ではなく `<img>` タグの使用（自動最適化の欠如）の検出
- `next/link` ではなく `<a>` タグの使用（プリフェッチの欠如）の検索
- 重いライブラリをインポートする `"use client"` ファイルによる大きなクライアントバンドルの特定

#### セキュリティ
- サニタイズなしの `dangerouslySetInnerHTML` 使用の確認
- サーバー専用であるべき `NEXT_PUBLIC_` 環境変数の露出の検出
- APIルートハンドラでの入力バリデーションとサニタイズの検証

### Step 3: Next.js固有の調査

Next.js App Router規約に固有のチェック:

#### メタデータ
- `metadata` または `generateMetadata` エクスポートが欠如した `page.tsx` と `layout.tsx` ファイルの一覧化
- Open Graphやdescriptionメタデータが欠如したページのフラグ付け

#### エラーとローディングの境界
- `error.tsx` が欠如しているルートセグメントの一覧化（特にデータ取得を行うセグメント）
- 非同期操作に対する `loading.tsx` やSuspense境界が欠如しているルートセグメントの一覧化
- `error.tsx` ファイルに `"use client"` ディレクティブとリセット機構が含まれていることの確認

#### クライアント vs サーバーコンポーネントのバランス
- すべての `"use client"` ファイルを一覧化し、各ファイルが本当にクライアントサイドのインタラクティビティを必要とするか評価
- 静的コンテンツのレンダリングやデータの受け渡しのみを行う `"use client"` コンポーネントのフラグ付け
- 不必要にクライアントバンドルに取り込まれている大きなコンポーネントツリーの確認

#### Propsドリリング
- 5つ以上のpropsを受け取るコンポーネントの特定（ドリリングの兆候）
- 3階層以上にまたがるpropチェーンの確認
- 共有ステートのパターンが見られる箇所でのReact Contextやカスタムフックの提案

#### Suspense境界
- Suspenseラッパーが欠如した非同期Server Componentの確認
- ローディング状態のないコンポーネント内の `fetch` 呼び出しの特定
- 低速データソースに対するストリーミングパターンの使用検証

### Step 4: レポート生成

以下のフォーマットで調査結果を出力する:

```
## Technical Debt Audit Report — Next.js

Project: <package.jsonのプロジェクト名>
Scan date: <YYYY-MM-DD>
Files scanned: <件数>

### HIGH Priority (<件数> items)

1. **[<カテゴリ>]** `path/to/file.ts:L<行番号>`
   **Finding:** <問題の説明>
   **Recommendation:** <修正方法>

### MEDIUM Priority (<件数> items)
...

### LOW Priority (<件数> items)
...

### Summary

| Category | HIGH | MEDIUM | LOW |
|---|---|---|---|
| Code Duplication | - | - | - |
| Architecture & Layering | - | - | - |
| Error Handling | - | - | - |
| Type Safety | - | - | - |
| Dead Code | - | - | - |
| Constants & Configuration | - | - | - |
| Component Size | - | - | - |
| Dependency Management | - | - | - |
| Testing | - | - | - |
| Accessibility | - | - | - |
| Performance | - | - | - |
| Security | - | - | - |
| Metadata | - | - | - |
| Error/Loading Boundaries | - | - | - |
| Client vs Server Balance | - | - | - |
| Props Drilling | - | - | - |
| Suspense Boundaries | - | - | - |
| **Total** | **X** | **X** | **X** |
```

#### 優先度判定基準

| 優先度 | 基準 |
|---|---|
| **HIGH** | セキュリティリスク、データ損失の可能性、本番障害、クリティカルパスでのエラー境界の欠如 |
| **MEDIUM** | 保守性・可読性の著しい低下、パフォーマンスへの影響、型安全性の欠如 |
| **LOW** | ベストプラクティスからの逸脱、コード品質の改善提案、外観的な問題 |

### Step 5: Issue作成

レポート提示後、どの検出項目をGitHub Issueとして作成するかユーザーに確認する:

1. すべての検出項目を番号付きリストで表示し、以下のプロンプトを表示する:
   ```
   Issue化する項目を番号で指定してください（例: 1,3,5 / all / none）
   ```
2. ユーザーの回答に基づいて対応する:
   - **`none`**: スキルを終了
   - **`all`**: すべての検出項目に対してIssueを作成
   - **特定の番号**: 選択された検出項目のみIssueを作成
3. 選択された各検出項目について、`/git-issue-create` の規約に従いIssueを作成する:
   - **タイトル**: 日本語、検出内容の簡潔な説明
   - **ラベル**: `enhancement` + レポートの優先度に対応する優先度ラベル（`HIGH` → `priority: high`、`MEDIUM` → `priority: medium`、`LOW` → `priority: low`）
   - **本文**: カテゴリ、ファイルパス、検出内容の詳細、改善案を含める
4. 作成されたIssueの番号とURLの一覧を表示する
