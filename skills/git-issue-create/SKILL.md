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

   ```markdown
   ## Overview
   <Describe the purpose of the change in 1-2 sentences>

   ## Background & Motivation
   <Why is this change needed>

   ## Implementation Approach
   <Technical approach overview>

   ## Acceptance Criteria
   - [ ] <criteria 1>
   - [ ] <criteria 2>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

   **For `documentation` / `chore`:**

   ```markdown
   ## Overview
   <Describe the purpose of the change in 1-2 sentences>

   ## Background & Motivation
   <Why is this change needed>

   ## Acceptance Criteria
   - [ ] <criteria 1>
   - [ ] <criteria 2>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   ```

### Step 3: Preview and User Confirmation

Display a preview in the following format and obtain user approval:

```text
## Issue Preview

**Title**: <title>
**Type Label**: <label>
**Priority Label**: <label>

---
<body>
---

Shall I create the Issue with this content? Let me know if you'd like any changes.
```

- If the user approves → proceed to Step 4
- If the user requests changes → apply the changes and display the preview again

### Step 4: Create Issue

1. Create the Issue using `gh issue create`:

   ```bash
   gh issue create --title "<title>" --label "<type label>" --label "<priority label>" --body "$(cat <<'EOF'
   <body>
   EOF
   )"
   ```

2. If it fails, display the error message and exit

### Step 5: Display Results

Display the creation results in the following format:

```text
## Issue Created

Issue #XX: <title>
<Issue URL>

- Type Label: <type label>
- Priority Label: <priority label>
```
