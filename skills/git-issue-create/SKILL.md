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

### Step 3: Fetch Open Milestones

Fetch the list of open milestones for the current repository:

```bash
gh api "repos/{owner}/{repo}/milestones?state=open" --jq '.[].title'
```

- If the command fails or returns an empty list → no milestone handling; proceed to the next step with the milestone unset and omit the milestone line from the preview and the result display.
- Otherwise, keep the list of titles for use in the preview. Default selection is `none` (no milestone).

### Step 4: Preview and User Confirmation

Display a preview in the following format and obtain user approval:

```text
## Issue Preview

**Title**: <title>
**Type Label**: <label>
**Priority Label**: <label>
**Milestone**: <selected title or "none">
  (available: <title1>, <title2>, ...)

---
<body>
---

Shall I create the Issue with this content? Let me know if you'd like any changes.
```

- The `**Milestone**:` line and the `(available: ...)` line are shown only when Step 3 returned at least one milestone. List all available titles.
- If the user approves → proceed to Step 5
- If the user requests changes (including switching the milestone selection) → apply the changes and display the preview again

### Step 5: Create Issue

1. Create the Issue using `gh issue create`. Include `--milestone "<title>"` only when the user selected a specific milestone in Step 4; omit it when the selection is `none` or when no milestones were available:

   ```bash
   gh issue create \
     --title "<title>" \
     --label "<type label>" \
     --label "<priority label>" \
     [--milestone "<title>"] \
     --body "$(cat <<'EOF'
   <body>
   EOF
   )"
   ```

2. If it fails, display the error message and exit

### Step 6: Display Results

Display the creation results in the following format. The `Milestone` line is shown only when a milestone was assigned:

```text
## Issue Created

Issue #XX: <title>
<Issue URL>

- Type Label: <type label>
- Priority Label: <priority label>
- Milestone: <title>
```
