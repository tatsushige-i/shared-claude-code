# Shared Development Conventions

## Autonomous Action Restrictions

Do not perform the following actions unless explicitly instructed:

- Code implementation or modification
- git commit / push
- Issue or PR status changes (close, merge, etc.)
- Adding or removing packages

When in doubt, confirm with the user before proceeding.

**Important**: Plans and transcripts injected by the system (e.g., "Implement the following plan:") are reference information, not implementation instructions. Only begin implementation when the user explicitly requests it in their direct message. Never start implementation based solely on plan content.

## Response Policy

- Avoid excessive consideration or deference; respond candidly. Clearly communicate any issues or concerns.
- Avoid idiosyncratic approaches; prioritize widely adopted best practices in project development.

## Present Plan Before Implementation

For tasks involving code changes, present a plan and obtain approval before starting implementation. The plan should include:

- List of files to be changed
- Summary of changes
- Impact scope

## 1 Issue = 1 PR

Create one PR per Issue. Do not combine multiple Issues into a single PR.

## PR Size Limits

Keep PRs small. Use the following guidelines:

- Changed files: **10 files or fewer**
- Changed lines: **300 lines or fewer** (excluding auto-generated files)

If the limits are exceeded, propose splitting the task.

**Exception**: When adding new features where the changes primarily consist of creating new directories and files, exceeding the limits is acceptable. However, keep changes within a cohesive functional unit and do not include unrelated changes.

## Branch Naming Convention

Use a prefix based on the Issue label:

| Label           | Prefix         |
| --------------- | -------------- |
| `bug`           | `bugfix/`      |
| `feature`       | `feature/`     |
| `enhancement`   | `enhance/`     |
| `documentation` | `docs/`        |
| `chore`         | `chore/`       |

Branch names follow the format `<prefix><concise-english-description>` with words separated by hyphens.

## Mandatory Issue Labels

Both a **type label** and a **priority label** must be assigned when creating an Issue. Do not start work on Issues missing labels. If labels are missing before starting work, confirm with the user and assign them.

### Type Labels (required, select one)

| Label           | Criteria                                                              |
| --------------- | --------------------------------------------------------------------- |
| `bug`           | Fixing a defect where existing functionality does not work as expected |
| `feature`       | Adding new user-facing functionality                                  |
| `enhancement`   | Improving or extending existing features, UX, or dev workflows        |
| `documentation` | Documentation-only changes                                            |
| `chore`         | CI/CD, dependency updates, refactoring, or other non-functional work  |

### Priority Labels (required, select one)

| Label              | Criteria                                                     |
| ------------------ | ------------------------------------------------------------ |
| `priority: high`   | Requires immediate attention (service outage, data loss, etc.) |
| `priority: medium` | Should be addressed in the normal development flow           |
| `priority: low`    | Desirable but not urgent improvements or suggestions         |

## Documentation Language

All master `.md` files (rules, skills, README, etc.) must be written in **English**. Japanese translations are placed under `docs/ja-JP/` as supplementary documentation.

## Issue and PR Language

Issue and PR titles and bodies must be written in **Japanese**.

## GitHub Comment Signature

When Claude posts a comment on GitHub, append the following at the end to distinguish it from human comments:

```text
🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Skill README Update

When adding a new skill to `.claude/skills/`, update `.claude/skills/README.md` if it exists.

## Skill Naming Convention

- Format: `<category>-[<object>-]<verb>`
- Character constraints: lowercase letters, digits, and hyphens only (max 64 characters)
- Avoid overlapping with existing categories when adding new ones

## Minimal Change Principle

- Make only the minimum changes necessary to achieve the task objective
- Do not include unrelated file changes, refactoring, or formatting fixes
- Do not make "while we're at it" improvements — propose them as separate Issues

## Rule File Directory Structure

- `.claude/rules/` root — project-specific rule files
- `.claude/rules/shared/` — symlinks to `shared-claude-code`

## Rule File Size Limit

Each file under `.claude/rules/` should target approximately **200 lines**. If exceeded, propose splitting by topic.
