---
name: git-pr-create
description: Create a GitHub PR from the current branch - analyze changes, check size limits, and generate PR with proper formatting
---

# Create PR Skill

Automate the workflow for creating a GitHub PR from the current branch. Performs Issue identification, PR size checking, diff analysis, and PR creation in a single flow.

## Steps

### Step 1: Check Prerequisites

1. Get the current branch with `git branch --show-current`
   - If on `main` → display "エラー: mainブランチ上ではPRを作成できません。作業ブランチに切り替えてください。" and exit
2. Check for existing PRs with `gh pr list --head <branch name> --json number,url,state`
   - If an `OPEN` PR exists → display "エラー: このブランチには既にオープンなPRがあります: <URL>" and exit
3. Check for uncommitted changes with `git status --porcelain`
   - If changes exist → analyze the changes, stage related files individually with `git add <file path>`, generate an appropriate commit message and `git commit` (do not use `git add -A` or `git add .`; for untracked files, judge relevance to the changes and exclude unrelated ones)
4. Verify commits exist with `git log main..HEAD --oneline`
   - If no commits → display "エラー: mainブランチからのコミットがありません。" and exit
5. Push the branch to remote:
   - Run `git push -u origin <branch name>`
   - If it fails, display the error and exit

### Step 2: Identify Related Issue

Search for the Issue automatically in the following order (do not ask the user):

1. Get commit messages with `git log main..HEAD --format=%s%n%b` and look for Issue numbers using the `#(\d+)` pattern
2. Search from the branch name — use `gh issue list --search "<keyword>" --json number,title,state` with numbers or keywords from the branch name
3. Get recent open Issues with `gh issue list --state open --limit 10 --json number,title,labels` and infer the Issue from relevance to the branch name and changes
4. Branch based on the result:
   - **Identified**: Verify with `gh issue view <number> --json number,title,state` and display the title. If verification fails (Issue doesn't exist or is closed), continue without an Issue
   - **Multiple candidates**: Auto-select with the following priority — (1) Issue number directly included in commit messages (2) Branch name keyword matches the title (3) Most recent Issue
   - **Not identified**: Continue without an Issue (do not ask)

### Step 3: PR Size Check

1. Measure changed files, added lines, and deleted lines with `git diff --numstat main...HEAD`
2. Exclude auto-generated files (UI library generated files, etc.) from line counts
3. Compare against the shared development conventions limits (10 files / 300 lines):
   - If exceeded → display a warning with a task-splitting suggestion and continue (do not request confirmation)
   - If changes primarily consist of creating new files/directories, note this and state that the exception rule applies

### Step 4: Diff Analysis

1. Understand the changes with `git log main..HEAD --oneline` and `git diff main...HEAD --stat`
2. If needed, review detailed diffs with `git diff main...HEAD`
3. Infer the PR type from the branch prefix:

   | Prefix         | PR Title Prefix |
   |----------------|-----------------|
   | `bugfix/`      | `fix: `         |
   | `feature/`     | `feat: `        |
   | `enhance/`     | `enhance: `     |
   | `docs/`        | `docs: `        |
   | `chore/`       | `chore: `       |

### Step 5: Documentation Consistency Check

From the diff file list in `git diff main...HEAD --name-only`, detect changes that may require documentation updates using the following generic heuristics:

1. **Detection patterns**:
   - **Route additions**: New additions of files that define routes by framework convention, such as `page.tsx`, `page.jsx`, `page.ts`, `page.js`, `route.tsx`, `route.ts` (extract new files only with `git diff main...HEAD --diff-filter=A --name-only`)
   - **Skill additions**: File additions/changes under `.claude/skills/`
   - **Config file changes**: Changes to config files at the project root (`*.config.*`, `.*rc`, `.*rc.*`, `tsconfig*.json`, `scripts` section in `package.json`, etc.)

2. **Consistency check targets**: When detected, verify the existence and content consistency of:
   - `README.md` — whether new features, routes, or config changes are reflected
   - `.claude/skills/README.md` — whether the skill list is updated when skills are added
   - `.claude/rules/` directory — whether rule-related changes are reflected

3. **Handling based on results**:
   - **Inconsistency detected**: Display a warning in the following format and continue (do not block PR creation):
     ```
     ⚠️ ドキュメント整合性チェック:
     - <検出内容>: <対象ドキュメント>の更新が必要な可能性があります
     ```
   - **No detection patterns matched**: Proceed to Step 6 without displaying anything

### Step 6: Create PR

1. Generate a PR title (70 characters or less, using the prefix inferred in Step 4)
2. Generate the PR body using the following template:

   ```
   ## Summary
   - <変更の要点1>
   - <変更の要点2>

   Closes #XX  ← Issue特定時のみ

   ## Test plan
   - [ ] <テスト項目>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

3. Create the PR with `gh pr create --base main --title "..." --body "..."`
   - Use a heredoc for the body to preserve formatting:
     ```
     gh pr create --base main --title "<タイトル>" --body "$(cat <<'EOF'
     <本文>
     EOF
     )"
     ```
4. If it fails, display the error message and exit

### Step 7: Display Results

Display the creation results in the following format:

```
## PR作成完了

PR #XX: <タイトル>
<PR URL>

- 対応Issue: #XX <Issueタイトル>  ← Issue特定時のみ
- 変更ファイル数: X件
- 変更行数: +XX / -XX
```
