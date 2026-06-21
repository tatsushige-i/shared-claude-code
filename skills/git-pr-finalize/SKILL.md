---
name: git-pr-finalize
description: Finalize a GitHub PR end-to-end - watch CI and Copilot review, address findings, merge after confirmation, and clean up branches
argument-hint: "[PR number]"
---

# PR Finalize Skill

Orchestrate the post-PR workflow end-to-end: watch CI checks and Copilot review, address findings, merge after user confirmation, and clean up branches.

This skill is an **orchestrator**. It does not re-implement comment handling or branch cleanup; it reuses existing skills:

- **`git-review-respond`** — analyze review comments, fix code, commit, and reply
- **`git-branch-cleanup`** — worktree-scoped local cleanup (`main`/`develop` protected)

The skill focuses on **monitoring + completion judgment + merge** only.

## Steps

### Step 1: Determine PR Number

- If `$ARGUMENTS` is specified as a number, use that PR number
- If `$ARGUMENTS` is empty (no argument), infer the PR from the current branch:
  - Run `gh pr view --json number,state,headRefName,title,url` (resolves the PR associated with the current branch)
  - If a PR is found, announce it and use that PR number:

    ```text
    Using PR #XX associated with the current branch `<branch>`.
    ```

  - If no PR is found (the command fails, or the current branch has no PR), fall back to asking the user for the PR number:

    ```text
    Could not detect a PR from the current branch. Please provide the PR number you'd like to finalize.
    ```

- If `$ARGUMENTS` is specified but is not a number, ask the user for the PR number:

  ```text
  Please provide the PR number you'd like to finalize.
  ```

- If a clear number cannot be determined from the user's response, exit with an error. Do not guess or make ambiguous interpretations:

  ```text
  Error: Could not identify the PR number. Please specify a number.
  ```

### Step 2: Fetch PR Information and Switch Branch

1. Fetch PR information with `gh pr view <PR number> --json state,headRefName,number,title,url,baseRefName,mergeable,mergeStateStatus`
2. If the command fails (no PR exists for the given number):
   - Check if it exists as an Issue with `gh issue view <PR number>`, and if it's an Issue, display the following and exit:

     ```text
     Error: #XX is an Issue. Please specify a PR number.
     ```

   - If it's not an Issue either, display the following and exit:

     ```text
     Error: PR #XX does not exist.
     ```

3. Check the PR's `state`:
   - If not `OPEN` → warn with "This PR is already closed/merged." and exit
4. Checkout the PR's `headRefName` branch:
   - Run `git checkout <headRefName>`
   - Pull the latest from remote with `git pull`

### Step 3: Monitoring Loop (CI + Copilot Review)

Monitor CI checks and Copilot review together. Repeat the loop until **both** are settled (CI all passing and no unresolved review threads), or a stop condition is hit.

1. **CI checks**: Poll the PR's checks with `gh pr checks <PR number>` (you may use `gh pr checks <PR number> --watch` to block until checks settle).
   - Treat `pending` / `in_progress` as "still running" — keep waiting.
   - When no running checks remain, classify the result as **all passing** or **failure** (any check in a failing/error state).

2. **Copilot / human review**: Fetch review state via GraphQL. The base is the same query as `git-review-respond` Step 3, extended with `reviewRequests` and `reviews` so the loop can confirm Copilot review **completion** before reading threads:

   ```graphql
   query {
     repository(owner: "{owner}", name: "{repo}") {
       pullRequest(number: <PR number>) {
         reviewRequests(first: 10) {
           nodes {
             requestedReviewer {
               ... on Bot { login }
               ... on User { login }
             }
           }
         }
         reviews(first: 50) {
           nodes {
             state
             author { login }
             submittedAt
           }
         }
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

   - **Gate on Copilot review completion first.** Before evaluating `reviewThreads`, classify the Copilot review state. An empty `reviewThreads` is **not** sufficient to conclude "no findings" — Copilot may simply not have posted yet:

     | State | Detection | Action |
     |---|---|---|
     | Copilot review pending | `reviewRequests` contains `copilot-pull-request-reviewer` but `reviews.nodes` has no Copilot entry | Keep waiting (continue the loop) |
     | Copilot review complete | `reviews.nodes` contains a Copilot entry with `submittedAt` set | Proceed to evaluate `reviewThreads` normally |
     | Copilot not configured | Neither `reviewRequests` nor `reviews` contains Copilot | Ask the user whether to proceed without Copilot review |

   - **Only after the gate passes**, detect unresolved threads: "Unresolved finding" = a `reviewThread` with `isResolved == false`. This includes Copilot (`copilot-pull-request-reviewer`) and human review comments.
   - Completion judgment uses **the presence of unresolved threads** (not re-review requests).

3. **Polling interval / timeout**:
   - CI typically takes a few minutes; Copilot review typically arrives within a few minutes. Poll at roughly **30–60 second** intervals.
   - Apply a maximum wait (e.g., **~15 minutes** total) per monitoring phase. On timeout, display the current CI and review state and stop — do not merge.

4. Branch on the loop result:
   - **Copilot review pending** (gate not yet passed) → keep waiting (continue the loop until completion or timeout).
   - **CI failure** → go to Step 4 (CI failure handling).
   - **Unresolved review threads present** → go to Step 4 (review finding handling).
   - **CI all passing AND Copilot review settled (complete, or not configured and confirmed by the user) AND no unresolved threads** → go to Step 5.

### Step 4: Address Findings (Conditional)

#### 4A: Review findings present

- Run the `git-review-respond` skill for this PR (`/git-review-respond <PR number>`). It handles analysis, code fixes, commit, push, and per-comment replies, following its own user-confirmation step.
- After it completes, return to **Step 3** and re-monitor (new commits trigger CI again, and replies/resolutions update thread state).

#### 4B: CI failure

- Inspect the failing job logs (e.g., `gh run view <run id> --log-failed`, or follow the check details URL from `gh pr checks`).
- Present the failure cause and a fix approach to the user.
- Apply the minimum fix, stage related files individually with `git add <file path>` (do not use `git add .`), commit, and `git push`.
- After pushing, return to **Step 3** and re-monitor.

#### Stop conditions

Stop and report (do not loop indefinitely) when:

- A CI failure cannot be resolved automatically.
- The PR is in a non-mergeable state (`mergeable == CONFLICTING`, merge conflicts, or a blocking `mergeStateStatus`).
- The monitoring timeout from Step 3 is reached.

Display the current state clearly and let the user decide next steps.

### Step 5: Completion Judgment

Confirm that **CI is all passing** and **there are no unresolved review threads**. Only then proceed to merge.

### Step 6: Merge (User Confirmation Required)

Merging is a destructive, outward-facing action. **Always confirm with the user before merging** (consistent with the shared conventions: PR status changes require explicit instruction).

1. Present a pre-merge summary and wait for explicit approval:

   ```text
   ## Ready to Merge

   PR #XX: <title>
   <PR URL>

   - CI: all passing
   - Review threads: all resolved
   - Base: <baseRefName>

   Merge this PR? (the head branch will be deleted)
   ```

2. After approval, merge with `gh pr merge <PR number> --delete-branch`.
   - The merge method follows the repository's settings (e.g., squash/merge/rebase as configured). Do not force a method unless the user specifies one.
   - `--delete-branch` removes the remote head branch. The base (`main`/`develop`) is never deleted.
3. If the merge fails (e.g., not mergeable, branch protection), display the error and stop.

### Step 7: Local Cleanup

Run the `git-branch-cleanup` skill (`/git-branch-cleanup`) to clean up the local worktree and working branch.

- It handles both the primary clone and linked worktrees, and protects `main`/`develop`.
- It resolves the return target from the merged PR's `baseRefName`, so it returns to whichever branch the PR targeted.

### Step 8: Completion Summary

Display the results in the following format:

```text
## PR Finalized

PR #XX: <title>
<PR URL>

- Merge: <merged | not merged (reason)>
- Findings addressed: <summary, e.g., "2 review rounds, 1 CI fix" or "none">
- Remote branch: <deleted | n/a>
- Local cleanup: <summary from git-branch-cleanup>
```
