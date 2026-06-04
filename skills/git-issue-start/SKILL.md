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

  ```text
  対応するIssueの番号を教えてください。
  ```

- If a clear number cannot be determined from the user's response, exit with an error. Do not guess or make ambiguous interpretations:

  ```text
  エラー: Issue番号を特定できませんでした。数値で指定してください。
  ```

### Step 2: Fetch Issue Information

1. Fetch Issue information with `gh issue view <Issue number> --json number,title,body,labels,state`
2. If the command fails (no Issue exists for the given number):
   - Check if it exists as a PR with `gh pr view <Issue number>`, and if it's a PR, display the following and exit:

     ```text
     エラー: #XX はPRです。Issueの番号を指定してください。
     ```

   - If it's not a PR either, display the following and exit:

     ```text
     エラー: Issue #XX は存在しません。
     ```

3. If `state` is not `OPEN` → warn with "このIssueは既にクローズされています" and exit
4. Display the fetched information in the following format:

   ```text
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

### Step 4: Determine Base Branch and Prefix

This skill auto-detects the base branch so it works for both `main`-only repositories (backward compatible) and `main` + `develop` (git-flow lite) repositories.

1. Detect whether `origin` has a `develop` branch:
   - Run `git ls-remote --heads origin develop`
   - Non-empty output → `develop` exists; empty → it does not

2. Determine the base branch and prefix:

   | `develop` on origin | Issue has `hotfix` label | Base branch | Prefix                              |
   |---------------------|--------------------------|-------------|-------------------------------------|
   | No                  | (any)                    | `main`      | from the type-label table below     |
   | Yes                 | Yes                      | `main`      | `hotfix/`                           |
   | Yes                 | No                       | `develop`   | from the type-label table below     |

   When `develop` does not exist, behavior is unchanged from before: base is `main` and a `hotfix` label has no effect.

3. Type-label prefix table (used unless `hotfix/` applies), based on the "Branch Naming Convention" in the shared development conventions (`conventions.md`):

   | Label           | Prefix         |
   |-----------------|----------------|
   | `bug`           | `bugfix/`      |
   | `feature`       | `feature/`     |
   | `enhancement`   | `enhance/`     |
   | `documentation` | `docs/`        |
   | `chore`         | `chore/`       |

4. Generate a branch name from the Issue title:
   - If the title is in Japanese, convert it to English
   - Use kebab-case (lowercase English separated by hyphens)
   - Format: `<prefix><concise-description>`
5. Decide the work location based on the existing clone's **current branch**
   (`git branch --show-current`). This prevents disturbing a clone that another
   parallel agent is already working in:

   - **Current branch is `main` or `develop` (the clone is idle)**: create the
     branch directly in the existing clone (unchanged from before):
     - Switch to the base branch: `git checkout <base>`
     - Pull latest: `git pull --ff-only`
     - Create and checkout branch: `git checkout -b <branch name>`

   - **Current branch is anything else (the clone is occupied by other in-flight
     work)**: do **not** touch the existing clone. Create an isolated git worktree
     from `<base>` instead:
     - If the project provides a worktree helper script, prefer it
       (e.g. `scripts/dev/create_worktree.sh <issue-number> <branch name> <base>`).
     - Otherwise: `git fetch origin`, then
       `git worktree add ../<repo>-<issue-number> -b <branch name> origin/<base>`.
     - The current session's cwd remains the original clone, so tell the user to
       **open the new worktree directory and continue work there** (re-run the
       skill / enter Plan Mode from that worktree). If the project documents a
       parallel worktree workflow, point the user to it.

   Backward compatibility: repositories whose clone always sits on `main` (or that
   have no `develop`) always take the idle path, so behavior is unchanged for them.

### Step 5: Project-Specific Scaffold (Conditional)

If scaffold procedures (file generation patterns for new features, etc.) are defined in the project's `.claude/rules/architecture.md`, follow those procedures to generate files.

If not defined, or if the Issue's type label is not a scaffold target, skip this step.

### Step 6: Transition to Plan Mode

1. Call the `EnterPlanMode` tool to transition to Plan Mode
2. Display the following message to prompt implementation planning:

   ```text
   Plan Modeに移行しました。Issue #XX の実装計画を策定します。

   ## Issue情報
   - タイトル: <タイトル>
   - ラベル: <ラベル一覧>

   <Issue本文>

   上記のIssue内容に基づいて実装計画を策定します。

   **実装完了後の注意**: 実装が完了したら `git diff --stat` および主要な差分をユーザーに提示し、問題がなければ `/git-pr-create` でPRを作成するよう案内すること。コミットは `/git-pr-create` のフローに委ねる。
   ```
