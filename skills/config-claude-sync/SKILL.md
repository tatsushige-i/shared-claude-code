---
name: config-claude-sync
description: Sync shared rules and skills from shared-claude-code repository - detect missing symlinks and create them
---

# Shared Config Sync Skill

Sync shared rules and skills from the shared-claude-code repository to the current project. Detect missing symlinks and create them after user confirmation.

## Steps

### Step 1: Locate the Shared Repository

1. Search for symlinks under `.claude/rules/shared/` and `.claude/skills/`
2. Resolve the shared-claude-code repository path from the `readlink` result of found symlinks
   - Rule link example: `../../../shared-claude-code/rules/conventions.md` → extract `shared-claude-code` path
   - Skill link example: `../../shared-claude-code/skills/git-pr-create` → extract `shared-claude-code` path
3. If no symlinks are found → display error and exit:
   ```
   Error: No symlinks to shared-claude-code found.
   Initial setup must be done manually following the README instructions.
   ```
4. Verify that `rules/` and `skills/` directories exist at the resolved path
   - If they do not exist → display error and exit

### Step 2: Detect Differences

1. Get the list of `.md` files under `rules/` in shared-claude-code
2. Get the list of directories containing `SKILL.md` under `skills/` in shared-claude-code
3. Compare with the current project and detect missing items:
   - **Rules**: no symlink exists at `.claude/rules/shared/<name>.md`
   - **Skills**: no symlink exists at `.claude/skills/<name>`

### Step 3: Present Differences and Confirm with User

1. If everything is already synced → display the following and exit:
   ```
   All rules and skills are already synced.
   ```
2. If there are missing items, display in the following format:
   ```
   ## Unsynced Items

   ### Rules
   - conventions.md
   - security.md

   ### Skills
   - git-branch-cleanup
   - git-issue-create

   Would you like to sync these items? Let me know if you want to exclude any.
   ```
3. Branch based on the user's response:
   - Full approval → sync all items
   - Partial exclusion → exclude the specified items and sync the rest

### Step 4: Create Branch

1. Check the current branch with `git branch --show-current`
2. **If not on main** → skip this step and proceed to Step 5 (assume working on an existing branch)
3. **If on main**:
   - Check for uncommitted changes with `git status --porcelain`
     - If changes exist → display error and exit:
       ```
       Error: There are uncommitted changes on the main branch. Please commit or stash the changes and run again.
       ```
   - Check if a branch with the same name exists with `git branch --list chore/sync-claude-rules`
     - If it does not exist → create branch with `git checkout -b chore/sync-claude-rules`
     - If it exists → create branch with `git checkout -b chore/sync-claude-rules-YYYYMMDD` (current date)

### Step 5: Create Symlinks

1. Rule sync:
   - Create `.claude/rules/shared/` directory with `mkdir -p` if it does not exist
   - Get the prefix from the `readlink` result of existing rule symlinks and create new symlinks using the same pattern
   - Example: if an existing link is `../../../shared-claude-code/rules/conventions.md`, create new ones as `../../../shared-claude-code/rules/<name>.md`
2. Skill sync:
   - Get the prefix from the `readlink` result of existing skill symlinks and create new symlinks using the same pattern
   - Example: if an existing link is `../../shared-claude-code/skills/git-pr-create`, create new ones as `../../shared-claude-code/skills/<name>`
3. After creating each symlink, verify that the link target resolves correctly

### Step 6: Update Documentation

If new skills were synced in Step 5, update documentation files that list skills. Skip this step if only rules were synced.

1. Check if `.claude/skills/README.md` exists in the current project
   - If it exists:
     - For each synced skill, check whether the skill name already appears in the file
     - For skills not yet listed, read the `description` from the `SKILL.md` frontmatter in the shared-claude-code `skills/<name>/SKILL.md`
     - Append a table row at the end of the existing table: `| \`<name>\` | \`/<name>\` | <description> |`
     - Stage the file with `git add .claude/skills/README.md`
   - If it does not exist: skip
2. Check if `docs/ja-JP/skills/README.md` exists in the current project
   - If it exists:
     - Apply the same process: check for missing entries and append table rows for unregistered skills
     - Stage the file with `git add docs/ja-JP/skills/README.md`
   - If it does not exist: skip
3. Check if `CLAUDE.md` in the project root contains a skills listing (e.g., a table or list mentioning existing skill names)
   - If it contains a skills listing:
     - For each synced skill not yet listed, append an entry matching the existing format
     - Stage the file with `git add CLAUDE.md`
   - If `CLAUDE.md` does not exist or does not contain a skills listing: skip

### Step 7: Commit

1. Stage the symlinks created in Step 5 individually (do not use `git add -A` or `git add .`):
   - Rules: `git add .claude/rules/shared/<name>.md`
   - Skills: `git add .claude/skills/<name>`
   - README files staged in Step 6 are already included
2. Commit with `git commit -m "chore: sync shared claude rules and skills"`
3. If the commit fails (e.g., no staged files), display a warning and proceed to Step 8

### Step 8: Display Results

Display sync results in the following format:

```
## Sync Complete

- Branch: <branch name> (newly created / existing)
- Commit: <short commit hash>
- Synced rules: X
  - <filename 1>
  - <filename 2>
- Synced skills: X
  - <skill name 1>
  - <skill name 2>
- Updated README: <list of updated files>

You can create a PR with `/git-pr-create`.
```

- Display "You can create a PR with `/git-pr-create`." only when a new branch was created in Step 4
- If running on a branch other than main, display "existing" and omit the PR suggestion
- Display "Updated README" only when README files were updated in Step 6
