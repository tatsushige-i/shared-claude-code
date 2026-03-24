---
name: config-github-sync
description: Sync .github files and ci-templates from shared-claude-code repository - detect language, present diffs, and copy files
---

# GitHub Config Sync Skill

Sync shared assets under `.github/` (ISSUE_TEMPLATE, PULL_REQUEST_TEMPLATE, workflows) from the shared-claude-code repository to the current project using file copies. File copies are used because GitHub does not recognize symlinks.

## Steps

### Step 1: Locate the Shared Repository

1. Search for symlinks under `.claude/rules/shared/` and `.claude/skills/`
2. Resolve the shared-claude-code repository path from the `readlink` result of found symlinks
   - Rule link example: `../../../shared-claude-code/rules/conventions.md` → extract `shared-claude-code` path
   - Skill link example: `../../shared-claude-code/skills/git-pr-create` → extract `shared-claude-code` path
3. If no symlinks are found → display error and exit:
   ```
   エラー: shared-claude-codeへのシンボリックリンクが見つかりません。
   最初のセットアップはREADMEの手順に従って手動で行ってください。
   ```
4. Verify that the `.github/` directory exists at the resolved path
   - If it does not exist → display error and exit

### Step 2: Detect Differences

1. Scan `.github/ISSUE_TEMPLATE/`, `.github/PULL_REQUEST_TEMPLATE.md`, and `.github/workflows/` on the shared side
2. Compare with the corresponding local files and determine each file's status:
   - **Identical**: contents match (determined by `diff -q`)
   - **Differs**: exists on both sides but contents differ
   - **New**: exists on the shared side but not locally
   - **Local only**: exists locally but not on the shared side (display warning only, do not delete)

### Step 3: Present Differences and Confirm with User

1. If all files are identical → display the following and exit:
   ```
   .github/ 配下のファイルはすべて同期済みです。
   ```
2. If differences exist, display status by category:
   ```
   ## .github 同期チェック

   ### ISSUE_TEMPLATE
   - bug.yml — 同一 ✓
   - chore.yml — ローカルに存在しない（新規追加）
   - enhancement.yml — 差分あり

   ### PULL_REQUEST_TEMPLATE
   - PULL_REQUEST_TEMPLATE.md — 差分あり

   ### workflows
   - ci.yml — 差分あり

   同期する項目を選択してください（例: 全て / ISSUE_TEMPLATEのみ）。
   差分の詳細を確認したい場合はお知らせください。
   ```
3. If the user requests diff details → show the `diff` output
4. Determine sync targets based on the user's response:
   - Full approval → sync all items
   - Partial exclusion → exclude the specified items and sync the rest

### Step 4: Copy Files

1. Create `.github/ISSUE_TEMPLATE/` and `.github/workflows/` directories with `mkdir -p` if they do not exist. For `PULL_REQUEST_TEMPLATE.md`, ensure `.github/` exists.
2. Overwrite-copy selected items with `cp`
3. After copying, verify contents match with `diff -q`
   - If mismatch → display error

### Step 5: CI Template Sync

1. Check whether the `ci-templates/` directory exists in the shared repository. If it does not exist, skip this step.
2. **Detect the project language/framework** from the local project root:
   - `package.json` present → Node.js family. If `dependencies.next` or `devDependencies.next` is found → **Next.js**
   - `pyproject.toml` or `setup.py` present → **Python**
   - `go.mod` present → **Go**
   - List available template directories under `ci-templates/` as additional candidates
3. **Present candidates and confirm with the user**:
   ```
   ## CI テンプレート同期

   推定フレームワーク: Next.js（package.json の dependencies.next を検出）

   利用可能なテンプレート:
   - nextjs（推奨）

   同期するテンプレートを選択してください（スキップする場合は「スキップ」と入力）。
   ```
   - If no matching template is found → display available templates and ask the user to choose or skip
4. **Detect differences** for the selected template:
   - `ci-templates/<lang>/.github/workflows/ci.yml` vs local `.github/workflows/ci.yml`
   - Each file directly under `ci-templates/<lang>/` (non-`.github/` items) vs the same filename at the local project root
   - Categorize each file as **Identical**, **Differs**, **New**, or **Local only** (same criteria as Step 2)
5. **Present differences and confirm with the user**:
   - If all files are identical → display "CI テンプレートのファイルはすべて同期済みです。" and skip copying
   - If differences exist, display by category:
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
   - If the user requests diff details → show `diff` output
   - Determine sync targets based on the user's response
6. **Copy files**:
   - Create destination directories with `mkdir -p` if they do not exist
   - Overwrite-copy selected files with `cp`
   - After copying, verify contents match with `diff -q`
     - If mismatch → display error

### Step 6: Label Color Sync

Standard label colors are defined as follows (source of truth: shared-claude-code repository):

| Label name | Color code |
|---|---|
| `bug` | `d73a4a` |
| `feature` | `0e8a16` |
| `enhancement` | `a2eeef` |
| `documentation` | `0075ca` |
| `chore` | `ededed` |
| `priority: high` | `b60205` |
| `priority: medium` | `fbca04` |
| `priority: low` | `0e8a16` |

1. Get the target repository's current labels with `gh label list --json name,color`
2. For each label in the standard table above, check if it exists in the target repository:
   - **Not found in target**: Skip (do not create)
   - **Color differs**: Flag as needing update
   - **Color matches**: Mark as `✓`
3. If all found labels match → display the following and skip to Step 7:
   ```
   All label colors match the standard.
   ```
4. If differences exist, display and ask for confirmation:
   ```
   ## Label Color Check

   - `bug`: d73a4a → e11d48 (differs)
   - `feature`: 0e8a16 ✓
   - `priority: high`: b60205 ✓

   Apply color corrections? (yes / skip)
   ```
5. If the user approves → run `gh label edit <name> --color <hex>` for each differing label
6. If the user skips → proceed to Step 7 without changes

### Step 7: Display Results

Display sync results in the following format:

```
## 同期完了

- 同期したISSUE_TEMPLATE: X件
  - <ファイル名1>（新規追加）
  - <ファイル名2>（更新）
- 同期したPULL_REQUEST_TEMPLATE: （新規追加 or 更新）
- 同期したワークフロー: X件
  - <ファイル名1>（更新）
- 同期したCIテンプレート（<lang>）: X件
  - .github/workflows/ci.yml（更新）
  - tsconfig.json（新規追加）
- Label colors corrected: X
  - `bug`: e11d48 → d73a4a
```

Omit the CI template line if CI templates were not synced. Omit the label color line if no label colors were corrected.
