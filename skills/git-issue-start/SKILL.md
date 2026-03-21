---
name: git-issue-start
description: Start implementation workflow from a GitHub Issue - fetch, validate labels, create branch, and enter plan mode
argument-hint: "<Issue number>"
---

# Implement Issue Skill

Automate the workflow from GitHub Issue information retrieval, label validation, branch creation, to Plan Mode transition.

## Steps

### Step 1: Determine Issue Number

- If `$ARGUMENTS` is specified as a number, use that Issue number
- If not specified or not a number, ask the user for the Issue number:
  ```
  対応するIssueの番号を教えてください。
  ```
- If a clear number cannot be determined from the user's response, exit with an error. Do not guess or make ambiguous interpretations:
  ```
  エラー: Issue番号を特定できませんでした。数値で指定してください。
  ```

### Step 2: Fetch Issue Information

1. Fetch Issue information with `gh issue view <Issue number> --json number,title,body,labels,state`
2. If the command fails (no Issue exists for the given number):
   - Check if it exists as a PR with `gh pr view <Issue number>`, and if it's a PR, display the following and exit:
     ```
     エラー: #XX はPRです。Issueの番号を指定してください。
     ```
   - If it's not a PR either, display the following and exit:
     ```
     エラー: Issue #XX は存在しません。
     ```
3. If `state` is not `OPEN` → warn with "このIssueは既にクローズされています" and exit
4. Display the fetched information in the following format:
   ```
   ## Issue #XX: <タイトル>

   ラベル: <ラベル一覧>

   <本文>
   ```

### Step 3: Label Validation

Validate the following based on the "Mandatory Issue Labels" section in the shared development conventions (`conventions.md`):

1. **Type label**: Verify that one of `bug`, `feature`, `enhancement`, `documentation`, `chore` is assigned
2. **Priority label**: Verify that one of `priority: high`, `priority: medium`, `priority: low` is assigned
3. If either is missing:
   - Inform the user which label type is missing and ask which label to assign
   - Assign it with `gh issue edit <Issue number> --add-label "<label>"` based on the user's response
4. If both are present, proceed to the next step

### Step 4: Create and Checkout Branch

1. Determine the prefix from the type label based on the "Branch Naming Convention" in the shared development conventions (`conventions.md`):

   | Label           | Prefix         |
   |-----------------|----------------|
   | `bug`           | `bugfix/`      |
   | `feature`       | `feature/`     |
   | `enhancement`   | `enhance/`     |
   | `documentation` | `docs/`        |
   | `chore`         | `chore/`       |

2. Generate a branch name from the Issue title:
   - If the title is in Japanese, convert it to English
   - Use kebab-case (lowercase English separated by hyphens)
   - Format: `<prefix><concise-description>`
3. Create the branch directly with the generated name:
   - Switch to main branch: `git checkout main`
   - Pull latest: `git pull`
   - Create and checkout branch: `git checkout -b <branch name>`

### Step 5: Project-Specific Scaffold (Conditional)

If scaffold procedures (file generation patterns for new features, etc.) are defined in the project's `.claude/rules/architecture.md`, follow those procedures to generate files.

If not defined, or if the Issue's type label is not a scaffold target, skip this step.

### Step 6: Transition to Plan Mode

1. Call the `EnterPlanMode` tool to transition to Plan Mode
2. Display the following message to prompt implementation planning:
   ```
   Plan Modeに移行しました。Issue #XX の実装計画を策定します。

   ## Issue情報
   - タイトル: <タイトル>
   - ラベル: <ラベル一覧>

   <Issue本文>

   上記のIssue内容に基づいて実装計画を策定します。
   ```
