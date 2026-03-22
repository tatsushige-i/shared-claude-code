---
name: git-issue-create
description: Create a GitHub Issue from conversation context - analyze context, generate title/body/labels, preview, and create
---

# Create Issue Skill

Automate the workflow for creating GitHub Issues from conversation context. Performs context analysis, title/body generation, label inference, preview confirmation, and Issue creation in a single flow.

## Steps

### Step 1: Context Analysis

1. Identify the subject and purpose to be turned into an Issue from the conversation context
2. Based on the "Mandatory Issue Labels" section in the shared development conventions (`conventions.md`), infer the following:
   - **Type label**: Select one based on the following criteria

     | Label           | Criteria                                                              |
     |-----------------|-----------------------------------------------------------------------|
     | `bug`           | Fixing a defect where existing functionality does not work as expected |
     | `feature`       | Adding new user-facing functionality                                  |
     | `enhancement`   | Improving or extending existing features, UX, or dev workflows        |
     | `documentation` | Documentation-only changes                                            |
     | `chore`         | CI/CD, dependency updates, refactoring, or other non-functional work  |

   - **Priority label**: Select one based on the following criteria

     | Label              | Criteria                                                     |
     |--------------------|--------------------------------------------------------------|
     | `priority: high`   | Requires immediate attention (service outage, data loss, etc.) |
     | `priority: medium` | Should be addressed in the normal development flow           |
     | `priority: low`    | Desirable but not urgent improvements or suggestions         |

### Step 2: Generate Issue Content

1. **Title**: Generate in Japanese, targeting around 50 characters
2. **Body**: Generate in Japanese using the template appropriate for the type label:

   **For `feature` / `enhancement` / `bug`:**
   ```
   ## 概要
   <1-2文で変更の目的を説明>

   ## 背景・動機
   <なぜこの変更が必要か>

   ## 実装方針
   <技術的なアプローチの概要>

   ## 受け入れ条件
   - [ ] <条件1>
   - [ ] <条件2>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

   **For `documentation` / `chore`:**
   ```
   ## 概要
   <1-2文で変更の目的を説明>

   ## 背景・動機
   <なぜこの変更が必要か>

   ## 受け入れ条件
   - [ ] <条件1>
   - [ ] <条件2>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

### Step 3: Preview and User Confirmation

Display a preview in the following format and obtain user approval:

```
## Issueプレビュー

**タイトル**: <タイトル>
**種類ラベル**: <ラベル>
**優先度ラベル**: <ラベル>

---
<本文>
---

この内容でIssueを作成してよろしいですか？修正点があればお知らせください。
```

- If the user approves → proceed to Step 4
- If the user requests changes → apply the changes and display the preview again

### Step 4: Create Issue

1. Create the Issue using `gh issue create`:
   ```
   gh issue create --title "<タイトル>" --label "<種類ラベル>" --label "<優先度ラベル>" --body "$(cat <<'EOF'
   <本文>
   EOF
   )"
   ```
2. If it fails, display the error message and exit

### Step 5: Display Results

Display the creation results in the following format:

```
## Issue作成完了

Issue #XX: <タイトル>
<Issue URL>

- 種類ラベル: <種類ラベル>
- 優先度ラベル: <優先度ラベル>
```
