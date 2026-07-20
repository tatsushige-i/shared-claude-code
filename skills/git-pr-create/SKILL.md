---
name: git-pr-create
description: Create a GitHub PR from the current branch - analyze changes, check size limits, and generate PR with proper formatting
argument-hint: "[--copilot-review] [--finalize]"
---

# Create PR Skill

Automate the workflow for creating a GitHub PR from the current branch. Performs Issue identification, PR size checking, diff analysis, and PR creation in a single flow.

## Steps

### Step 0: Parse Flags and Resolve Base Branch

Resolve the base branch first so the rest of the flow works for `main`-only repositories (backward compatible), `main` + `develop` (git-flow lite) repositories, and repositories with `release/*` branches (standard Git Flow). Use the resolved value wherever `<base>` appears below.

1. Parse `$ARGUMENTS` for optional flags (can be combined, order-independent):
   - `--copilot-review`: request Copilot as a reviewer when the PR is created (used in Step 6)
   - `--finalize`: automatically continue into the `git-pr-finalize` flow after PR creation (used in Step 7)
   - If neither flag is present, behavior is unchanged (backward compatible)
   - State explicitly which flags were detected (e.g. "Flags: --copilot-review=no, --finalize=yes") before continuing, and treat only a flag confirmed present in `$ARGUMENTS` as active in Step 6 / Step 7 — never add `--reviewer @copilot` or chain into `git-pr-finalize` by default
2. Get the current branch with `git branch --show-current`
3. Determine `<base>` (first matching rule wins):
   - If the current branch starts with `hotfix/` → `<base>` = `main`
   - Else, detect a `release/*` branch the current branch was cut from: list release branches with `git ls-remote --heads origin 'release/*'`. If any exist, run `git fetch origin` (so their tips are available locally), then for each release branch `R` test `git merge-base --is-ancestor origin/<R> HEAD`. Among the release branches that are ancestors of `HEAD` (the current branch was cut from them), pick the one whose merge-base with `HEAD` is the most recent (the closest ancestor) → `<base>` = that release branch
   - Else → run `git ls-remote --heads origin develop`; if `develop` exists → `<base>` = `develop`, else → `<base>` = `main`

Backward compatibility: when no `release/*` branch is an ancestor of `HEAD` and no `develop` exists, `<base>` falls back to `main`, unchanged from before.

### Step 1: Check Prerequisites

1. Get the current branch with `git branch --show-current`
   - If on `main` or `develop` → display "Error: Cannot create a PR on the `main`/`develop` branch. Please switch to a working branch." and exit
2. Check for existing PRs with `gh pr list --head <branch name> --json number,url,state`
   - If an `OPEN` PR exists → display "Error: This branch already has an open PR: <URL>" and exit
3. Check for uncommitted changes with `git status --porcelain`
   - If changes exist → analyze the changes, stage related files individually with `git add <file path>`, generate an appropriate commit message and `git commit` (do not use `git add -A` or `git add .`; for untracked files, judge relevance to the changes and exclude unrelated ones)
4. Verify commits exist with `git log <base>..HEAD --oneline`
   - If no commits → display "Error: No commits from the `<base>` branch." and exit
5. Push the branch to remote:
   - Run `git push -u origin <branch name>`
   - If it fails, display the error and exit

### Step 2: Identify Related Issue

Search for the Issue automatically in the following order (do not ask the user):

1. Get commit messages with `git log <base>..HEAD --format=%s%n%b` and look for Issue numbers using the `#(\d+)` pattern
2. Search from the branch name — use `gh issue list --search "<keyword>" --json number,title,state` with numbers or keywords from the branch name
3. Get recent open Issues with `gh issue list --state open --limit 10 --json number,title,labels` and infer the Issue from relevance to the branch name and changes
4. Branch based on the result:
   - **Identified**: Verify with `gh issue view <number> --json number,title,state` and display the title. If verification fails (Issue doesn't exist or is closed), continue without an Issue
   - **Multiple candidates**: Auto-select with the following priority — (1) Issue number directly included in commit messages (2) Branch name keyword matches the title (3) Most recent Issue
   - **Not identified**: Continue without an Issue (do not ask)

### Step 3: PR Size Check

1. Measure changed files, added lines, and deleted lines with `git diff --numstat <base>...HEAD`
2. Exclude auto-generated files (UI library generated files, etc.) from line counts
3. Compare against the shared development conventions limits (10 files / 300 lines):
   - If exceeded → display a warning with a task-splitting suggestion and continue (do not request confirmation)
   - If changes primarily consist of creating new files/directories, note this and state that the exception rule applies

### Step 4: Diff Analysis

1. Understand the changes with `git log <base>..HEAD --oneline` and `git diff <base>...HEAD --stat`
2. If needed, review detailed diffs with `git diff <base>...HEAD`
3. Infer the PR type from the branch prefix:

   | Prefix         | PR Title Prefix |
   |----------------|-----------------|
   | `bugfix/`      | `fix:`         |
   | `hotfix/`      | `fix:`         |
   | `feature/`     | `feat:`        |
   | `enhance/`     | `enhance:`     |
   | `docs/`        | `docs:`        |
   | `chore/`       | `chore:`       |

   Note on base: `hotfix/` targets `main` (urgent production fixes); a branch cut from a `release/*` branch targets that release branch (release preparation / reject handling); otherwise `bugfix/` and the other prefixes target `develop` when it exists (otherwise `main`). The base was already resolved in Step 0.

### Step 5: Documentation Consistency Check

From the diff file list in `git diff <base>...HEAD --name-only`, detect changes that may require documentation updates using the following generic heuristics:

1. **Detection patterns**:
   - **Route additions**: New additions of files that define routes by framework convention, such as `page.tsx`, `page.jsx`, `page.ts`, `page.js`, `route.tsx`, `route.ts` (extract new files only with `git diff <base>...HEAD --diff-filter=A --name-only`)
   - **Skill additions**: File additions/changes under `.claude/skills/`
   - **Config file changes**: Changes to config files at the project root (`*.config.*`, `.*rc`, `.*rc.*`, `tsconfig*.json`, `scripts` section in `package.json`, etc.)

2. **Consistency check targets**: When detected, verify the existence and content consistency of:
   - `README.md` — whether new features, routes, or config changes are reflected
   - `.claude/skills/README.md` — whether the skill list is updated when skills are added
   - `.claude/rules/` directory — whether rule-related changes are reflected

3. **Handling based on results**:
   - **Inconsistency detected**: Display a warning in the following format and continue (do not block PR creation):

     ```text
     ⚠️ Documentation Consistency Check:
     - <detection>: <target document> may need to be updated
     ```

   - **No detection patterns matched**: Proceed to Step 6 without displaying anything

### Step 6: Create PR

1. Generate a PR title (70 characters or less, using the prefix inferred in Step 4)
2. Generate the PR body using the following template:

   ```markdown
   ## Summary
   - <key change 1>
   - <key change 2>

   Closes #XX  ← only when Issue is identified

   ## Test plan
   - [ ] <test item>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

3. Create the PR with `gh pr create --base <base> --title "..." --body "..."`
   - If `--copilot-review` was specified, add `--reviewer @copilot` (uses `gh` v2.88.0+'s native Copilot reviewer support)
   - Use a heredoc for the body to preserve formatting:

     ```bash
     gh pr create --base <base> --title "<title>" --body "$(cat <<'EOF'
     <body>
     EOF
     )"
     ```

4. If it fails, display the error message and exit

### Step 7: Display Results

Display the creation results in the following format:

```text
## PR Created

PR #XX: <title>
<PR URL>

- Related Issue: #XX <Issue title>  ← only when Issue is identified
- Files changed: X
- Lines changed: +XX / -XX
```

If `--finalize` was specified, continue directly into the `git-pr-finalize` flow (run it with no arguments — it resolves the PR from the current branch). The merge confirmation gate in `git-pr-finalize` Step 6 is unchanged — this flag only chains the flow, it does not skip that confirmation.
