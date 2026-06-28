---
name: review-team-run
description: Run a parallel code-review team of stance-distinct subagents over the pre-PR diff (mainly uncommitted changes) and present a consolidated, severity-ordered report
---

# Review Team Run Skill

Review the pre-PR diff with a team of five stance-distinct review subagents launched **in parallel**, then consolidate their findings into one severity-ordered report.

This skill is meant to run **before** `/git-pr-create` — in shared usage `git-pr-create` handles commit-through-PR, so this skill typically reviews the **uncommitted working-tree changes** (plus any not-yet-PR'd commits on the branch). It is complementary to the built-in `/code-review`, not a replacement.

The five members and their primary domains (each pins its own model via frontmatter):

| Subagent | Model | Primary domain |
|---|---|---|
| `review-correctness` | opus | Correctness & bugs |
| `review-security` | opus | Security |
| `review-design` | opus | Design & simplicity |
| `review-readability` | sonnet | Readability & cleanliness |
| `review-tests` | sonnet | Testing |

Performance and accessibility are intentionally out of scope for this generic team.

## Steps

### Step 1: Resolve Base and Collect the Review Diff

1. Resolve `<base>` exactly as `git-pr-create` Step 0 does (first matching rule wins):
   - Current branch starts with `hotfix/` → `<base>` = `main`
   - Else, if a `release/*` branch on `origin` is an ancestor of `HEAD` → `<base>` = the closest such release branch (run `git fetch origin` first so their tips are local; test each with `git merge-base --is-ancestor origin/<R> HEAD`)
   - Else, if `git ls-remote --heads origin develop` is non-empty → `<base>` = `develop`
   - Else → `<base>` = `main`

2. Collect the **pre-PR delta** as the union of three sources (the changes that would become the PR, whether committed yet or not):
   - **Uncommitted tracked changes**: `git diff HEAD`
   - **Untracked files**: list with `git ls-files --others --exclude-standard`, then include each file's content (skip binary files and anything matched by `.gitignore`)
   - **Committed branch changes**: `git diff <base>...HEAD` (empty when there are no commits beyond `<base>`)

3. Build the file list and line counts for display with `git diff <base> --stat` combined with `git status --short` (so both committed and uncommitted changes are reflected). Exclude obvious auto-generated files from the headline counts.

4. If all three sources are empty → display `レビュー対象の差分がありません。` and exit.

### Step 2: Present the Target and the Team

Show what will be reviewed and by whom, then proceed (no confirmation needed — launching read-only reviewers is non-destructive):

```text
## Review Team Run

Target: <branch> vs <base> — <N> files, +<X> / -<Y> lines
（未コミット差分を含む）

Members (parallel):
- review-correctness (opus) — 正確性・バグ
- review-security (opus) — セキュリティ
- review-design (opus) — 設計・シンプルさ
- review-readability (sonnet) — 可読性・クリーンさ
- review-tests (sonnet) — テスト
```

### Step 3: Launch the Five Reviewers in Parallel

Issue **all five `Agent` calls in a single message** so they run concurrently. For each call set `subagent_type` to the member name (`review-correctness`, `review-security`, `review-design`, `review-readability`, `review-tests`).

Give every member the same diff payload and an identical framing, varying only the domain reminder:

- The collected diff from Step 1 (or, when the diff is large, the list of changed files plus the per-file diffs — the subagents have `Read`/`Grep`/`Glob` and can open surrounding code themselves).
- A one-line reminder of the member's primary domain and that it must report **only** findings in that domain.
- The output-format instruction below (each subagent's own prompt also restates it, so this reinforces it): emit each finding as a block, or exactly `No findings.` when there is nothing in the domain.

```text
- file:line: `path/to/file.ext:42`
  severity: critical | high | medium | low
  finding: <what is wrong and why it matters>
  fix: <the minimal concrete change>
  member: <the member name, e.g. review-security>
```

Severity: `critical` = breaks production / data loss / security breach (must fix before merge); `high` = real bug or risk likely in normal use (fix before merge); `medium` = meaningful maintainability or edge-case/test-gap issue; `low` = minor nit.

Wait for all five to return before continuing.

### Step 4: Consolidate Findings

1. Parse each member's per-finding blocks. Members returning `No findings.` contribute nothing.
2. **De-duplicate**: when two or more members report the same `file:line` with the same substance, merge into one entry and list all reporting members (e.g. `[review-correctness, review-design]`).
3. **Order** by severity: `critical` → `high` → `medium` → `low`. Within a severity, keep file order.
4. Build the member × severity summary table. A merged finding counts once per contributing member.

### Step 5: Present the Consolidated Report

Merge all members' findings (de-duplicating same `file:line` + same-substance findings, attributing each to its member) and output using this consolidated format:

```text
## Code Review — Team Result

Target: <branch> vs <base>, <N> files, +<X> / -<Y> lines

### CRITICAL (<count>)
1. `path/to/file.ext:42` — [review-security]
   <finding>
   → Fix: <fix suggestion>

### HIGH (<count>)
...

### MEDIUM (<count>)
...

### LOW (<count>)
...

### Summary

| Member | critical | high | medium | low |
|---|---|---|---|---|
| review-correctness | - | - | - | - |
| review-security | - | - | - | - |
| review-design | - | - | - | - |
| review-readability | - | - | - | - |
| review-tests | - | - | - | - |
| **Total** | **0** | **0** | **0** | **0** |
```

- Omit any severity section that has zero findings.
- If every member returned `No findings.`, display `指摘はありませんでした。` and the empty summary table.
- This skill **presents only** — it does not edit code, commit, or create Issues. Suggest the user address findings and then run `/git-pr-create` when ready.
