---
name: impl-pipeline
description: Drive Issue implementation with support agents — plan (read-only), split mechanical vs. interactive at a human gate, implement, test from spec, review with review-team-run, triage, fix, and stop at a final human gate before /git-pr-create.
argument-hint: "<Issue number>"
---

# Implementation Pipeline Skill

Orchestrate the post-planning implementation of a single GitHub Issue using a small team of support agents, while keeping every judgment call — the mechanical/interactive split, the review triage, and the go-ahead to commit — with the human (main session).

This skill runs **after `/git-issue-start`** (branch created, Plan Mode plan approved) and **hands off to `/git-pr-create`** at the end. It reuses the existing `/review-team-run` skill for the review phase.

## Design principles (do not violate)

- **The planner does not decide.** `impl-planner` presents the plan and the decision points; the human makes the mechanical-vs.-interactive call and every product/UX decision.
- **coder implements a confirmed plan only.** Autonomous implementation (`impl-coder`) runs only for the mechanical parts, after the human gate.
- **tester writes from the spec.** `impl-tester` derives tests from acceptance criteria / `docs/spec`, never from the just-written implementation code.
- **Review findings are not fixed unverified.** Triage `review-team-run` output by severity and fix only CONFIRMED / worthwhile findings.
- **No orchestrator agents, no agent nesting.** Orchestration is this skill's (main session's) job — do **not** create `impl-orchestrator` or `review-orchestrator` agents, and never have one agent launch another. Only skill→agent calls (this skill launches `impl-planner`/`impl-coder`/`impl-tester`; `/review-team-run` launches the review team).
- **Commit / PR stay with the human.** This skill never commits, pushes, or opens/merges PRs; it ends by pointing to `/git-pr-create`.

## Steps

### Step 1: Preconditions and Issue Context

1. Confirm the current branch is the Issue's working branch (not `main`/`develop`) and the Plan Mode plan is approved. If not, tell the user to run `/git-issue-start <Issue#>` first and stop.
2. Determine the Issue number from `$ARGUMENTS`; if absent, ask for it.
3. Fetch the Issue body and comments to pass to the planner:
   - `gh issue view <Issue number> --json number,title,body`
   - `gh issue view <Issue number> --comments`

### Step 2: Plan (impl-planner — every Issue)

Launch `Agent` with `subagent_type: impl-planner`, passing the Issue title/body/comments and any relevant spec paths (`docs/spec`, `.claude/rules/`). The planner is read-only and returns a plan plus **decision points** and a mechanical-vs.-interactive recommendation — it does not decide.

Present the returned plan to the user verbatim (or lightly summarized), including the decision points.

### Step 3: Human Gate ① — Mechanical vs. Interactive

Using the planner's decision points and recommendation, have the **user** decide how to proceed (use `AskUserQuestion`):

- **Mechanical** — implementable autonomously from the confirmed plan.
- **Interactive** — needs a prototype → screenshot → compare → decide loop, kept in the main session.

Resolve any open decision points with the user before implementing. Do not proceed until the split and the open decisions are settled.

### Step 4: Implement

- **Mechanical Issue** → launch `Agent` with `subagent_type: impl-coder`, passing the confirmed plan. The coder implements the plan, updates generated artifacts (e.g. `build_runner`), and formats. If it reports a blocker/uncovered decision, return to the user rather than guessing.
- **Interactive / UI Issue** → implement in the **main session** with the user (prototype → screenshot → compare → decide). Do **not** use `impl-coder` for the parts that need judgment.

A single Issue may split into mechanical parts (coder) and interactive parts (main session); route each part accordingly.

### Step 5: Test (impl-tester)

Launch `Agent` with `subagent_type: impl-tester`, passing the Issue's acceptance criteria, `docs/spec`, and any test-spec document — **not** the implementation as the source of truth. It implements and runs tests. If it reports a failing test that reveals a real defect, treat that as a finding for triage (Step 7); do not silently make the test pass.

### Step 6: Review (reuse review-team-run)

Run the existing `/review-team-run` skill over the pre-PR diff. Do **not** re-define the five reviewers here — this skill only consumes the consolidated, severity-ordered report they produce.

### Step 7: Triage (human, planner may assist)

Triage the review report (and any defect from Step 5) by severity. Adopt only **CONFIRMED / worthwhile** findings — `critical`/`high` should generally be fixed before PR; `medium`/`low` are judgment calls. You may ask `impl-planner` to help assess a finding, but the adopt/skip decision is the human's. Record which findings are adopted and which are intentionally skipped.

### Step 8: Fix → Re-test → CI Gate

For adopted findings:

1. Apply fixes with `impl-coder` (mechanical) or in the main session (judgment).
2. Re-run `impl-tester`.
3. Confirm the CI gates locally: formatter/linter clean and generated artifacts fresh (e.g. `build_runner` output up to date).

Repeat triage→fix until no adopted finding remains.

### Step 9: Human Gate ② — Final Go / PR

Present the final diff summary (`git diff --stat` and the key changes) and the adopted/skipped findings to the user. On the user's go-ahead, hand off to `/git-pr-create` (which owns commit-through-PR). This skill never commits or opens the PR itself.
