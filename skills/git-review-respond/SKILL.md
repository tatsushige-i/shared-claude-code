---
name: git-review-respond
description: Respond to GitHub PR review comments - analyze, fix code, and reply to each comment
argument-hint: "<PR number>"
---

# PR Review Comment Response Skill

Fetch and analyze GitHub PR review comments, then perform code fixes, commits, and replies in bulk.

## Steps

### Step 1: Determine PR Number

- If `$ARGUMENTS` is specified as a number, use that PR number
- If not specified or not a number, ask the user for the PR number:
  ```
  対応するPRの番号を教えてください。
  ```
- If a clear number cannot be determined from the user's response, exit with an error. Do not guess or make ambiguous interpretations:
  ```
  エラー: PR番号を特定できませんでした。数値で指定してください。
  ```

### Step 2: Fetch PR Information and Switch Branch

1. Fetch PR information with `gh pr view <PR number> --json state,headRefName,number,title,url`
2. If the command fails (no PR exists for the given number):
   - Check if it exists as an Issue with `gh issue view <PR number>`, and if it's an Issue, display the following and exit:
     ```
     エラー: #XX はIssueです。PRの番号を指定してください。
     ```
   - If it's not an Issue either, display the following and exit:
     ```
     エラー: PR #XX は存在しません。
     ```
3. Check the PR's `state`:
   - If not `OPEN` → warn with "このPRは既にクローズ/マージされています" and exit
4. Checkout the PR's `headRefName` branch:
   - Run `git checkout <headRefName>`
   - Pull the latest from remote with `git pull`

### Step 3: Fetch and Filter Review Comments

1. Fetch review comments via REST API:
   ```
   gh api repos/{owner}/{repo}/pulls/<PR number>/comments --paginate
   ```

2. Fetch resolved thread comment IDs via GraphQL API:
   ```graphql
   query {
     repository(owner: "{owner}", name: "{repo}") {
       pullRequest(number: <PR number>) {
         reviewThreads(first: 100) {
           nodes {
             isResolved
             comments(first: 100) {
               nodes {
                 databaseId
               }
             }
           }
         }
       }
     }
   }
   ```

3. Exclude the following comments:
   - Comments with `in_reply_to_id` set (reply comments)
   - Comments belonging to resolved threads

4. If 0 target comments remain → display "対応すべきレビューコメントはありません" and exit

### Step 4: Analyze and Classify Each Comment

For each target comment, review the following information:
- `path`: Target file path
- `line` / `original_line`: Target line
- `diff_hunk`: Diff context
- `body`: Comment body

#### Validity Assessment Criteria

Before classification, critically evaluate each comment's content against the following criteria:

1. **Actual harm**: Does it point out a real bug, security risk, or performance issue? Items that are merely style suggestions or preference differences should lean toward "No action needed"
2. **Project consistency**: Is the suggestion consistent with the project's existing patterns and rules? Do not adopt suggestions that contradict project conventions
3. **Cost-benefit**: Is the value worth the added complexity? Reject suggestions for excessive defensive coding or unnecessary abstractions
4. **Automated review tool responses**: Evaluate suggestions from automated review tools (Copilot, etc.) with particular scrutiny. Automated tools lack project-specific context, so classify unnecessary items as "No action needed" without deference

After applying these criteria, classify each comment into one of the following:

| Classification     | Criteria                                                           | Action              |
|--------------------|--------------------------------------------------------------------|---------------------|
| **Needs fix**      | Requests code changes (fix requests, improvement suggestions, bug reports, etc.) | Code fix + reply    |
| **Needs response** | Questions, confirmations, intent clarifications, etc.              | Reply only          |
| **No action needed** | Praise, impressions, already addressed, etc.                     | Reply only (or no reply needed) |

**Note**: If the `path` file does not exist (deleted), skip that comment and record the reason.

### Step 5: Present Analysis Results (User Confirmation)

Display the classification results in the following format and wait for user approval:

```
## PRレビューコメント分析結果

PR #XX: <タイトル>
対象コメント: X件

### 要修正 (X件)
1. `path/to/file.ts:L42` - @reviewer
   > コメント内容の要約
   → 修正方針: ～を変更する

### 要説明 (X件)
1. `path/to/file.ts:L10` - @reviewer
   > コメント内容の要約
   → 回答方針: ～について説明する

### 対応不要 (X件)
1. `path/to/file.ts:L5` - @reviewer
   > コメント内容の要約
   → 理由: 称賛コメント

この内容で対応を進めてよいですか？
```

**Wait for user approval before proceeding to the next step.** If the user changes the classification or approach, follow their direction.

### Step 6: Code Fixes

After approval, implement code fixes for comments classified as "Needs fix".

- Read the target file, identify the indicated location, and apply the fix
- Fixes should be the minimum changes that address the comment's intent
- If the fix affects related files, fix those as well

### Step 7: Verification

Run the project's quality check commands (lint, test, etc.) to verify the fixes are sound.

If any check fails, fix the issue and re-run.

### Step 8: Commit and Push

1. Stage changed files individually with `git add <file path>` (do not use `git add .`)
2. Commit with the following format:
   ```
   fix: address PR #<PR number> review comments
   ```
   * Include Co-Authored-By
3. Push to remote with `git push`

**Note**: If there are no "Needs fix" comments and no code changes, skip this step.

### Step 9: Reply to Comments

Reply to each comment using `gh api`. Post replies in the same thread as the original comment:

```
gh api repos/{owner}/{repo}/pulls/<PR number>/comments \
  -method POST \
  -f body="<返信内容>" \
  -F in_reply_to=<元コメントのID>
```

Reply content guidelines:
- **Needs fix (fixed)**: Briefly explain the fix (e.g., "修正しました。`xxx` を `yyy` に変更しています。")
- **Needs response**: Write the answer to the question
- **No action needed**: Explain the reason if needed, or reply with thanks

Write replies in Japanese (when the reviewer uses Japanese). If the review comment is in English, reply in English.

**Signature**: Append the following signature at the end of each reply body:

```
🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Step 10: Completion Summary

Display the results in the following format:

```
## 完了サマリー

PR #XX: <タイトル>
<PR URL>

- 修正済み: X件
- 説明返信: X件
- 対応不要: X件
- スキップ: X件（理由: ファイル削除済みなど）

コミット: <コミットハッシュ>
```
