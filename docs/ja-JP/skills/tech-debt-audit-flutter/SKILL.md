---
name: tech-debt-audit-flutter
description: Flutter / Dart プロジェクトの技術的負債を調査し、優先度付きレポートを生成する
---

# Flutter 技術的負債調査スキル

Flutter / Dart プロジェクトの技術的負債を調査する。`rules/tech-debt-checklist.md` の各カテゴリを Flutter / Dart 固有の手順で調査し、フレームワーク固有のチェック（Widget 肥大化、ステート管理の一貫性、生成コードの鮮度等）も実施する。ファイルパス・行番号・改善案を含む優先度付きレポートを出力する。

## Steps

### Step 1: プロジェクト検出

1. リポジトリルートに `pubspec.yaml` が存在することを確認し、`flutter:` ブロックまたは `dependencies:` 配下の `flutter` SDK 制約を含むことで Flutter / Dart プロジェクトであることを検証する
   - `pubspec.yaml` が存在しない場合 → 「エラー: Dart プロジェクトではありません（pubspec.yaml が見つかりません）。」と表示して終了
   - `pubspec.yaml` は存在するが `flutter` 依存がない場合 → 「警告: 純粋な Dart パッケージのようです。Flutter 固有のチェック（Step 3 の Widget / State / Navigation）はスキップします。」と表示して続行
2. アプリケーションプロジェクトの場合は `lib/main.dart` の存在を確認する。`main.dart` を持たないライブラリパッケージも有効な調査対象とする
3. 後続コマンドで参照する Flutter CLI プレフィックスを決定する:
   - `.fvmrc` または `.fvm/` が存在する → `fvm flutter ...` / `fvm dart ...` を使用
   - それ以外 → `flutter ...` / `dart ...` を使用
4. プロジェクト構造をスキャンしてレイアウトを把握する:
   - `lib/` 配下のトップレベルディレクトリ（例: `app/`、`core/`、`features/`）を一覧化
   - 各 feature ディレクトリについて、クリーンアーキテクチャレイヤリング（`presentation/` / `application/` / `domain/` / `data/`）を採用しているか、その他の構造かを記録
   - `test/`、`integration_test/`、`analysis_options.yaml`、`.github/workflows/` の有無と内容を確認

### Step 2: 共通チェックリスト調査

`rules/tech-debt-checklist.md` の各カテゴリを Flutter / Dart コンテキストで調査する:

#### コードの重複

- 画面間で繰り返される Widget ツリーパターン（類似の `Scaffold` + `AppBar` + `ListView` の骨格、繰り返される `Padding` / `Container` のネスト等）の検索
- feature 間で重複するデータ取得や Repository 実装の確認
- `lib/core/` および `features/*/` 間で重複するユーティリティ関数の検索
- 共有 value object の不在を示唆する `freezed` モデルフィールドの重複検出

#### アーキテクチャとレイヤー分離

- feature-first / クリーンアーキテクチャレイヤリングを採用している場合、レイヤー違反を確認:
  - `presentation/` ファイルが `data/` を直接インポートしている（`application/` 経由とすべき）
  - `domain/` ファイルが Flutter、Riverpod、Drift 等のフレームワークをインポートしている
  - `application/` ファイルが `presentation/` のウィジェットをインポートしている
- feature 間の直接インポート（`features/foo/` が `features/bar/` をインポート）の確認 — 共有コードは `core/` に配置すべき
- 循環インポートチェーンの検出（疑わしい循環への手動 grep、または設定されていれば `dart run import_sorter` / `dart run dependency_validator`）
- `application/`（Riverpod notifier）に属すべきステートフルロジックが `StatefulWidget` の setState ブロックに漏れていないか確認

#### エラーハンドリング

- 失敗しうる Future（I/O、ネットワーク、プラグイン呼び出し）への `await` 呼び出しが `try / catch` の外側に存在していないか検索
- 空の `catch` ブロック（`catch (_) {}`）やスタックトレースを握り潰す広範すぎる `catch (e)` の確認
- `Future` / `Stream` コンシューマで `onError` が欠如しているケース、および失敗を無視する `unawaited()` 呼び出しの確認
- UI レイヤで `AsyncValue` コンシューマ（`ref.watch(...)`）の `error` 分岐欠如（`when` / `whenData` でエラーを落としている）の確認
- 型付きエラークラスとすべき `throw Exception('...')` 文字列の検索

#### 型安全性

- `.dart` ファイル全体での `dynamic` パラメータ / 戻り値型、および値の袋として使われる `Object?` の検索
- null 安全性を回避する null アサーション演算子（`!`）の使用フラグ付け
- 実行時に失敗しうる `as` キャストの検索
- 境界で `freezed` モデルへパースする代わりに複数レイヤを通過する `Map<String, dynamic>` の確認
- 初期化子の型が非自明な `var` 宣言での型注釈欠如のフラグ付け

#### デッドコード

- `<flutter-prefix> analyze` を実行し、`unused_element`、`unused_import`、`unused_local_variable`、`dead_code` の診断を収集
- コメントアウトされたコードブロック（コードに見える 3 行以上の連続コメント行）の検索
- 一度も再インポートされていない `export` 文の確認
- 現スプリントより古い TODO / FIXME / XXX コメント（放置作業の可能性）の特定

#### 定数と設定

- `build()` メソッド内のマジックナンバー（theme トークンや名前付き定数とすべき素のパディング / 半径 / 期間リテラル）の検索
- ソースファイルに埋め込まれたハードコード URL、API ベースパス、フィーチャーフラグ文字列の確認
- `Widget` コンストラクタで見落とされた `const` 機会（アナライザルール `prefer_const_constructors`）の確認
- ファイル間で重複する `Duration` / `EdgeInsets` / 色リテラル定義の特定

#### コンポーネント / モジュールの肥大化

- **300 行**を超える `.dart` ファイルの特定（生成ファイル `*.g.dart` / `*.freezed.dart` を除く）
- **100 行**を超える `build()` メソッドのフラグ付け — `extract widget` リファクタの候補
- **4 段階以上**ネストした単一の Widget ツリーのフラグ付け
- 無関係な責務を混在させた約 10 個以上の public メソッドを持つクラスのフラグ付け

#### 依存関係の管理

- `pubspec.yaml` の依存関係と `lib/` 内の実際の import を突き合わせて未使用パッケージを発見
- `<flutter-prefix> pub outdated` を実行し、メジャーバージョンが複数遅れているパッケージのフラグ付け
- 機能が重複するパッケージ（例: 2 つの HTTP クライアント、2 つの日付ライブラリ、2 つのモッキングフレームワーク）の検出
- バージョン固定を回避する git-ref や path 依存（`git:` / `path:` エントリ）のフラグ付け
- `test/`、`build_runner` 設定、CI のいずれからも参照されていない `dev_dependencies` エントリの確認

#### テスト

- `<flutter-prefix> test --coverage` を実行（または欠如を記録）し、`coverage/lcov.info` を確認して未カバーファイルを把握
- `lib/features/*/` のファイル数と `test/features/*/` のファイル数を比較して feature ごとのカバレッジギャップを推定
- ウィジェット / 統合テストが欠如しているクリティカルパス（認証、決済、永続化、同期）の特定
- インメモリ実装ではなく Repository をモックするテスト（統合バグを隠す可能性）の確認
- DI なしで実時刻の `DateTime.now()`、`Random()`、ファイルシステム状態に依存するテストのフラグ付け

#### アクセシビリティ

- `Semantics` ラベルなしで非インタラクティブコンテンツをラップする `GestureDetector` / `InkWell` の検索
- `Image`、`Icon`、`CircleAvatar` での `semanticLabel` / `semanticsLabel` 欠如の確認
- `tooltip` のない `IconButton` / `FloatingActionButton` の確認
- 色だけによる伝達（アイコンやラベルを伴わない赤 / 緑のテキスト等）のフラグ付け
- `TextField` での `labelText` / `hintText` セマンティクス欠如の確認

#### パフォーマンス

- `build()` 内の高コスト計算（ソート、JSON パース、正規表現コンパイル等）— `application/` でメモ化すべきものの確認
- 大量の子要素をハードコードした `ListView(...)` のフラグ付け — 可能な場合は `itemExtent` を指定した `ListView.builder` を使用すべき
- ホットパス（アニメーションティック、スクロールリスナー）での `setState` 呼び出しによる大規模サブツリーの再構築の検索
- 生成されたが破棄されない `StreamSubscription`、`TextEditingController`、`AnimationController`、`FocusNode` の特定
- ウィジェットツリー差分最適化を無効化するウィジェットコンストラクタの `const` 欠如の確認

#### セキュリティ

- 未検証のソースから取得する git / URL / path 依存の `pubspec.yaml` 内確認
- ソースに埋め込まれたシークレット、API キー、トークン（`String _apiKey = '...'`）の検索
- `flutter_secure_storage` / Keychain に置くべき機密データへの `SharedPreferences` 使用のフラグ付け
- SQL、ファイルパス、プラットフォームチャネルに直接渡される未検証の外部入力の確認
- ネットワーク呼び出しでハードコードされた `http://`（非 TLS）URL の確認

### Step 3: Flutter / Dart 固有の調査

共通チェックリストでカバーされない Flutter / Dart 規約固有のチェック。

#### Widget / build メソッドの肥大化

- 100 行を超える `build()` メソッドの一覧化
- 中間 `extract widget` リファクタなしに 4 階層以上ネストした Widget ツリーの特定
- 単一の `build()` 内でレイアウト・データ変換・イベントハンドリングを混在させている画面のフラグ付け

#### ステート管理の一貫性

- ステート管理アプローチの混在（同一プロジェクト内で `setState` + Riverpod + Provider + Bloc）の検出
- プロジェクトが Riverpod を使用している場合、`@riverpod` 生成プロバイダとすべき手書きの `Provider(...)` / `StateProvider(...)` のフラグ付け
- プロジェクトが Riverpod を使用している場合、notifier に置くべきビジネスステートを保持する `StatefulWidget` のフラグ付け
- ステート管理レイヤを迂回するグローバル可変シングルトン（トップレベルの `final foo = Foo();`）の確認

#### Async / Stream ハンドリング

- Riverpod の `AsyncValue` が一貫性のあるプロジェクトでの `FutureBuilder` / `StreamBuilder` 使用の特定
- `initState` で生成して `dispose` で `cancel()` していない未キャンセル `StreamSubscription` のフラグ付け
- 未破棄の `TextEditingController` / `AnimationController` / `FocusNode` / `ScrollController` のフラグ付け
- ウィジェット内の `Timer` および `Future.delayed` 使用での dispose 時キャンセル欠如の確認

#### 生成コードの鮮度

- `<flutter-prefix> dart run build_runner build --delete-conflicting-outputs` を実行し `git status` を確認
- いずれかの `*.g.dart` または `*.freezed.dart` ファイルが変更された → コミット済みの生成コードが古く再生成が必要
- 生成ファイルが（プロジェクトの規約に従って）リポジトリにコミットされており、ignore されていないことを検証

#### ナビゲーション規約

- プロジェクトが `go_router` を使用している場合、`context.push` / `context.go` / `context.pop` 経由とすべき直接的な `Navigator.push` / `Navigator.pop` / `Navigator.of(context)` 使用の検索
- ルート定義が一元化（例: `lib/app/router.dart`）されており、画面に散在していないことの確認
- ルートに宣言する代わりに画面内で手動パースされているディープリンクパラメータのフラグ付け

#### レイヤー境界の調査（プロジェクト固有）

- プロジェクトの `CLAUDE.md` または `.claude/rules/architecture.md` がレイヤリング規約を定義している場合、各 `lib/features/<feature>/` ディレクトリが規定の構造に従っているかを検証
- 規約が完全なレイヤーセットを要求している箇所でレイヤーが欠如している feature（例: `application/` のない `presentation/` のみ）のフラグ付け
- `core/` がガラクタ置き場として使われていないことの確認 — 単一 feature でしか使われないユーティリティはその feature 内に配置すべき

#### Build / CI カバレッジギャップ

- `.github/workflows/*.yml` を調査して PR で実行されるチェックを把握
- 重要と判断される欠如した CI ステップのフラグ付け:
  - `<flutter-prefix> format --output=none --set-exit-code .`（または `dart format`）
  - `<flutter-prefix> test --coverage` とカバレッジレポーター
  - 非ブロッキング助言としての `<flutter-prefix> pub outdated --exit-if-no-update-needed=false`
  - 生成コード鮮度チェック（`build_runner build` + `git diff --exit-code`）
- 最近リグレッションを引き起こした実績がない限り、これらは LOW 優先度とする

### Step 4: レポート生成

以下のフォーマットで調査結果を出力する:

```text
## Technical Debt Audit Report — Flutter

Project: <pubspec.yaml のプロジェクト名>
Scan date: <YYYY-MM-DD>
Files scanned: <件数>

### HIGH Priority (<件数> items)

1. **[<カテゴリ>]** `path/to/file.dart:L<行番号>`
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
| Component / Module Size | - | - | - |
| Dependency Management | - | - | - |
| Testing | - | - | - |
| Accessibility | - | - | - |
| Performance | - | - | - |
| Security | - | - | - |
| Widget / build Bloat | - | - | - |
| State Management Consistency | - | - | - |
| Async / Stream Handling | - | - | - |
| Generated Code Freshness | - | - | - |
| Navigation Conventions | - | - | - |
| Layer Boundary | - | - | - |
| Build / CI Coverage Gaps | - | - | - |
| **Total** | **X** | **X** | **X** |
```

#### 優先度判定基準

| 優先度 | 基準 |
|---|---|
| **HIGH** | セキュリティリスク、データ損失の可能性、本番障害、リーク原因となる未破棄リソース、クリティカルパスでの未処理エラー |
| **MEDIUM** | 保守性・可読性の著しい低下、パフォーマンスへの影響、型安全性の欠如、ステート管理の不一致 |
| **LOW** | ベストプラクティスからの逸脱、コード品質の改善提案、外観的な問題、助言レベルの CI ギャップ |

### Step 5: Issue 作成

レポート提示後、どの検出項目を GitHub Issue として作成するかユーザーに確認する:

1. すべての検出項目を番号付きリストで表示し、以下のプロンプトを表示する:

   ```text
   Issue化する項目を番号で指定してください（例: 1,3,5 / all / none）
   ```

2. ユーザーの回答に基づいて対応する:
   - **`none`**: スキルを終了
   - **`all`**: すべての検出項目に対して Issue を作成
   - **特定の番号**: 選択された検出項目のみ Issue を作成
3. 選択された各検出項目について、`/git-issue-create` の規約に従い Issue を作成する:
   - **タイトル**: 日本語、検出内容の簡潔な説明
   - **ラベル**: `enhancement` + レポートの優先度に対応する優先度ラベル（`HIGH` → `priority: high`、`MEDIUM` → `priority: medium`、`LOW` → `priority: low`）
   - **本文**: カテゴリ、ファイルパス、検出内容の詳細、改善案を含める
4. 作成された Issue の番号と URL の一覧を表示する
