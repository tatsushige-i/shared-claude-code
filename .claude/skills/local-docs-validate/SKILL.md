---
name: local-docs-validate
description: Validate documentation consistency across the repository - check skills, rules, symlinks, README structure, and translations
---

# Documentation Validation Skill

Validate documentation consistency across the shared-claude-code repository. Scan skills, rules, symlinks, README structure sections, and Japanese translations to detect inconsistencies. Offer automatic fixes where possible.

## Steps

### Step 1: Scan Skills

1. List all directories under `skills/` (excluding `README.md`)
2. For each skill directory, verify:
   - **1-1 SKILL.md exists with frontmatter**: `skills/<name>/SKILL.md` exists and contains `name` and `description` in YAML frontmatter
   - **1-2 Symlink exists**: `.claude/skills/<name>` exists as a symlink pointing to `../../skills/<name>`
     - Check with `readlink .claude/skills/<name>`
     - If the symlink does not exist or points to a different target → flag as issue
   - **1-3 README table entry**: `skills/README.md` contains a row with the skill name in the skills table
     - Check with `grep` for the skill name in `skills/README.md`
   - **1-4 Japanese translation**: `docs/ja-JP/skills/<name>/SKILL.md` exists
3. Record each issue found with its category and details

### Step 2: Scan Rules

1. List all `.md` files under `rules/` (excluding `README.md` if present)
2. For each rule file, verify:
   - **2-1 Japanese translation**: `docs/ja-JP/rules/<filename>` exists
3. Record each issue found

### Step 3: Validate Structure Sections

1. **README.md skills list**:
   - Parse the Structure section in `README.md` and extract listed skill names
   - Compare with actual directories under `skills/`
   - Flag skills that exist on disk but are missing from the Structure section
   - Flag entries in the Structure section that do not exist on disk
2. **Japanese README.md skills list**:
   - Parse the Structure section in `docs/ja-JP/README.md`
   - Compare with the English `README.md` Structure section
   - Flag any differences between the two lists

### Step 4: Check Core Translations

Verify that the following core translation files exist:

| # | File | Check |
|---|---|---|
| 4-1 | `docs/ja-JP/README.md` | File exists |
| 4-2 | `docs/ja-JP/CLAUDE.md` | File exists |
| 4-3 | `docs/ja-JP/skills/README.md` | File exists |

### Step 5: Present Findings

1. If no issues were found in any category → display:

   ```text
   All documentation consistency checks passed.
   ```

2. If issues were found → display results by category:

   ```text
   ## Documentation Consistency Check Results

   ### Skills Integrity
   - `config-github-sync` — symlink missing
   - `git-pr-create` — Japanese translation missing

   ### Rules Integrity
   - All passed

   ### Structure Sections
   - README.md — `config-github-sync` not listed in Structure section

   ### Core Translations
   - All passed

   Auto-fixable items are available. Apply fixes? (all / select / skip)
   ```

3. Wait for user response before proceeding

### Step 6: Apply Fixes

Based on the user's response, apply fixes:

**Auto-fixable** (apply with user approval):

- **Missing symlink**: Create with `ln -s ../../skills/<name> .claude/skills/<name>`, then verify with `readlink`
- **Missing skills/README.md entry**: Read `name` and `description` from `skills/<name>/SKILL.md` frontmatter, generate a table row in the format `| \`<name>\` | \`/<name>\` | <description> |`, and append it to the table in`skills/README.md`

**Manual fix required** (display warning only):

- **Structure section mismatch**: Display which files need updating and which skills are missing/extra
- **Missing Japanese translation**: Display which files need to be created

Display results after applying fixes:

```text
## Fixes Applied

- Created symlink: .claude/skills/config-github-sync
- Added README table entry: config-github-sync

Manual action required:
- Update Structure section in README.md to include `config-github-sync`
- Create Japanese translation: docs/ja-JP/skills/config-github-sync/SKILL.md
```
