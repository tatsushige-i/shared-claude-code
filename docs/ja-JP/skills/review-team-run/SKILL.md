---
name: review-team-run
description: スタンスの異なる subagent からなる並列コードレビューチームを PR作成前の差分（主に未コミット変更）に対して走らせ、重要度順の統合レポートを提示する
---

# Review Team Run スキル

PR作成前の差分を、スタンスの異なる5人のレビュー subagent を**並列**起動してレビューし、指摘を1つの重要度順レポートに統合する。

本スキルは `/git-pr-create` の**前**に走らせることを想定する — 共有運用では `git-pr-create` がコミット〜PR作成を担うため、本スキルは通常**未コミットの作業ツリー変更**（およびブランチ上の未PRコミット）をレビューする。組み込みの `/code-review` を置き換えるものではなく、補完するもの。

5人のメンバーと主担当（各自が frontmatter でモデルを固定）:

| Subagent | モデル | 主担当 |
|---|---|---|
| `review-correctness` | opus | 正確性・バグ |
| `review-security` | opus | セキュリティ |
| `review-design` | opus | 設計・シンプルさ |
| `review-readability` | sonnet | 可読性・クリーンさ |
| `review-tests` | sonnet | テスト |

パフォーマンスとアクセシビリティは、この汎用チームでは意図的に対象外とする。

## ステップ

### Step 1: base 解決とレビュー差分の収集

1. `git-pr-create` Step 0 と同じく `<base>` を解決する（最初に一致したルールを採用）:
   - 現在ブランチが `hotfix/` で始まる → `<base>` = `main`
   - そうでなく、`git ls-remote --heads origin 'release/*'` で `release/*` ブランチを列挙し、いずれかが `HEAD` の祖先なら → `<base>` = 最も近いもの（先に `git fetch origin` で tip をローカルに用意し、各 `R` を `git merge-base --is-ancestor origin/<R> HEAD` で判定、祖先のうち `HEAD` との merge-base が最新のものを選ぶ）
   - そうでなく、`git ls-remote --heads origin develop` が非空 → `<base>` = `develop`
   - それ以外 → `<base>` = `main`

2. **PR作成前デルタ**を収集する（コミット済みか否かによらず、PRに載る変更）:
   - **tracked 変更（コミット済み＋未コミット）**: `git diff <base>` — 作業ツリーを `<base>` と比較することで、ブランチのコミットと未コミットの作業ツリー編集を重複なく1つの差分で取得できる。
   - **未追跡ファイル**: `git ls-files --others --exclude-standard` で列挙し、各ファイルの内容を含める（バイナリと `.gitignore` 該当はスキップ）。

3. 表示用のファイル一覧と増減行は `git diff <base> --stat` に前ステップの未追跡ファイルを加えて構築する。明らかな自動生成ファイルは見出しの件数から除外する。

4. tracked 差分・未追跡リストの両方が空なら → `レビュー対象の差分がありません。` を表示して終了。

### Step 2: 対象とチームの提示

何を誰がレビューするかを示し、そのまま進める（read-only のレビュアー起動は非破壊なので確認不要）:

```text
## Review Team Run

Target: <branch> vs <base> — <N> files, +<X> / -<Y> lines
（未コミット差分を含む）

Members (parallel):
- review-correctness (opus) — 正確性・バグ
- review-security (opus) — セキュリティ
- review-design (opus) — 設計・シンプルさ
- review-readability (sonnet) — 可読性・クリーンさ
- review-tests (sonnet) — テスト
```

### Step 3: 5体を並列起動

**5つの `Agent` 呼び出しを1メッセージ内で**発行し、並列実行する。各呼び出しの `subagent_type` にメンバー名（`review-correctness`, `review-security`, `review-design`, `review-readability`, `review-tests`）を設定する。

全メンバーに同じ差分ペイロードと同一の枠組みを渡し、ドメインの念押しだけを変える:

- Step 1 で収集した差分（差分が大きい場合は、変更ファイル一覧＋ファイルごとの差分 — subagent は `Read`/`Grep`/`Glob` を持ち、周辺コードを自分で開ける）。
- メンバーの主担当の1行リマインダと、そのドメインの指摘**のみ**を報告すること。
- 以下の出力フォーマット指示（各 subagent 自身の prompt にも同じ記載があり、これはその補強）: 各指摘をブロックで出力、ドメインに何も無ければちょうど `No findings.`。

```text
- file:line: `path/to/file.ext:42`
  severity: critical | high | medium | low
  finding: <何が問題でなぜ重要か>
  fix: <最小限の具体的変更>
  member: <メンバー名、例: review-security>
```

重要度: `critical` = 本番破壊/データ損失/セキュリティ侵害（マージ前に必須修正）、`high` = 通常利用で表面化しうる実バグ/リスク（マージ前に修正）、`medium` = 有意な保守性やエッジケースの問題、`low` = 軽微な指摘。

5体すべての返却を待ってから続行する。

### Step 4: 指摘の統合

1. 各メンバーの指摘ブロックを解析する。`No findings.` を返したメンバーは寄与なし。
2. **重複排除**: 2人以上が同一 `file:line` を同趣旨で報告した場合、1エントリに統合し報告した全メンバーを併記する（例: `[review-correctness, review-design]`）。
3. **並べ替え**: 重要度順 `critical` → `high` → `medium` → `low`。同一重要度内はファイル順を保つ。
4. メンバー×重要度のサマリ表を構築する。統合された指摘は寄与メンバーごとに1件として数える。

### Step 5: 統合レポートの提示

全メンバーの指摘を統合（同一 `file:line` かつ同趣旨を重複排除し、各指摘を担当メンバーに帰属）し、以下の統合フォーマットで出力する:

```text
## Code Review — Team Result

Target: <branch> vs <base>, <N> files, +<X> / -<Y> lines

### CRITICAL (<count>)
1. `path/to/file.ext:42` — [review-security]
   <finding>
   → Fix: <fix suggestion>

### HIGH (<count>)
...

### MEDIUM (<count>)
...

### LOW (<count>)
...

### Summary

| Member | critical | high | medium | low |
|---|---|---|---|---|
| review-correctness | - | - | - | - |
| review-security | - | - | - | - |
| review-design | - | - | - | - |
| review-readability | - | - | - | - |
| review-tests | - | - | - | - |
| **Total** | **0** | **0** | **0** | **0** |
```

- 指摘0件の重要度セクションは省略する。
- 全メンバーが `No findings.` を返した場合は、`指摘はありませんでした。` と空のサマリ表を表示する。
- 本スキルは**提示のみ** — コード編集・コミット・Issue作成はしない。指摘に対応し、準備ができたら `/git-pr-create` を実行するようユーザーに促す。
