---
name: impl-pipeline
description: 支援エージェント群で Issue 実装を進める — プラン（読み取り専用）、人間ゲートで機械的 vs. 対話を切り分け、実装、仕様からテスト、review-team-run でレビュー、トリアージ、修正、そして /git-pr-create の前の最終人間ゲートで止まる。
argument-hint: "<Issue番号>"
---

# 実装パイプライン スキル

小さな支援エージェントチームを使って 1 つの GitHub Issue の実装（プラン策定後）をオーケストレーションする。一方で、あらゆる判断 — 機械的/対話の切り分け、レビューのトリアージ、コミットへの GO — は人間（メインセッション）に残す。

このスキルは **`/git-issue-start` の後**（ブランチ作成済み・Plan Mode のプラン承認済み）に実行し、最後に **`/git-pr-create` へ引き継ぐ**。レビューフェーズは既存の `/review-team-run` スキルを再利用する。

## 設計原則（違反しないこと）

- **プランナーは決めない。** `impl-planner` はプランと判断ポイントを提示する。機械的 vs. 対話の切り分けと、あらゆる product/UX 判断は人間が下す。
- **coder は確定プランのみ実装する。** 自律実装（`impl-coder`）は人間ゲートの後、機械的な部分に対してのみ走る。
- **tester は仕様から書く。** `impl-tester` は受け入れ基準／`docs/spec` からテストを導き、書きたての実装コードからは導かない。
- **レビュー指摘を無検証で直さない。** `review-team-run` の出力を severity でトリアージし、CONFIRMED / 価値ある指摘のみ修正する。
- **orchestrator エージェントを作らない、agent を入れ子にしない。** オーケストレーションはこのスキル（メインセッション）の責務 — `impl-orchestrator` や `review-orchestrator` エージェントを作らず、あるエージェントが別のエージェントを起動しない。skill→agent の呼び出しのみ（このスキルが `impl-planner`/`impl-coder`/`impl-tester` を起動し、`/review-team-run` がレビューチームを起動する）。
- **commit / PR は人間に残す。** このスキルは commit・push・PR の作成/マージをせず、最後に `/git-pr-create` を案内して終える。

## ステップ

### Step 1: 前提と Issue コンテキスト

1. 現在ブランチが Issue の作業ブランチであること（`main`/`develop` でない）と Plan Mode のプランが承認済みであることを確認する。そうでなければ `/git-issue-start <Issue#>` を先に実行するよう伝えて停止する。
2. `$ARGUMENTS` から Issue 番号を決定する。無ければ尋ねる。
3. プランナーに渡すため Issue 本文とコメントを取得する:
   - `gh issue view <Issue番号> --json number,title,body`
   - `gh issue view <Issue番号> --comments`

### Step 2: プラン（impl-planner — 全 Issue）

`Agent` を `subagent_type: impl-planner` で起動し、Issue のタイトル/本文/コメントと関連する仕様パス（`docs/spec`、`.claude/rules/`）を渡す。プランナーは読み取り専用で、プランに加えて **判断ポイント** と機械的 vs. 対話の推奨を返す — 決定はしない。

返ってきたプランを（判断ポイントを含めて）ユーザーにそのまま（または軽く要約して）提示する。

### Step 3: 人間ゲート①（機械的 vs. 対話）

プランナーの判断ポイントと推奨を踏まえ、進め方を **ユーザー** に決めてもらう（`AskUserQuestion` を使う）:

- **機械的** — 確定プランから自律実装できる。
- **対話** — 仮実装→スクショ→見比べ→決定のループを要し、メインセッションに保持する。

実装前に未解決の判断ポイントをユーザーと詰める。切り分けと未解決の判断が確定するまで進めない。

### Step 4: 実装

- **機械的 Issue** → `Agent` を `subagent_type: impl-coder` で起動し、確定プランを渡す。coder はプランを実装し、生成物（例: `build_runner`）を更新し、フォーマットする。ブロッカーや未カバーの判断を報告してきたら、推測せずユーザーに戻す。
- **対話 / UI Issue** → **メインセッション** でユーザーと実装する（仮実装→スクショ→見比べ→決定）。判断を要する部分に `impl-coder` を **使わない**。

1 つの Issue が機械的な部分（coder）と対話的な部分（メインセッション）に分かれることもある。各部分を適切に振り分ける。

### Step 5: テスト（impl-tester）

`Agent` を `subagent_type: impl-tester` で起動し、Issue の受け入れ基準・`docs/spec`・テスト仕様書を渡す — **実装を真実の源としない**。テストを実装・実行する。実バグを露呈する失敗テストを報告してきたら、それをトリアージ（Step 7）対象の指摘として扱う。黙ってテストを通さない。

### Step 6: レビュー（review-team-run を再利用）

既存の `/review-team-run` スキルを PR 作成前の差分に対して実行する。ここで 5 体のレビュアーを **再定義しない** — このスキルは彼らが生成する統合済み・severity 順のレポートを消費するだけ。

### Step 7: トリアージ（人間、planner が補助可）

レビューレポート（および Step 5 の欠陥）を severity でトリアージする。**CONFIRMED / 価値ある** 指摘のみ採択する — `critical`/`high` は概ね PR 前に修正、`medium`/`low` は判断次第。`impl-planner` に指摘の評価を手伝わせてよいが、採択/見送りの決定は人間のもの。採択した指摘と意図的に見送った指摘を記録する。

### Step 8: 修正 → 再テスト → CI ゲート

採択した指摘について:

1. `impl-coder`（機械的）またはメインセッション（判断）で修正を適用する。
2. `impl-tester` を再実行する。
3. CI ゲートをローカルで確認する: フォーマッタ/リンタがクリーンで、生成物が鮮度良い（例: `build_runner` 出力が最新）。

採択した指摘が無くなるまでトリアージ→修正を繰り返す。

### Step 9: 人間ゲート②（最終 GO / PR）

最終差分の要約（`git diff --stat` と主要な変更）と採択/見送りの指摘をユーザーに提示する。ユーザーの GO を受けて `/git-pr-create`（commit〜PR を担う）へ引き継ぐ。このスキル自身は commit も PR 作成もしない。
