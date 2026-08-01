---
name: tech-debt-audit-nextjs
description: Audit technical debt in Next.js (App Router) projects - analyze code against checklist and generate prioritized report
---

# Next.js Technical Debt Audit Skill

Audit a Next.js (App Router) project for technical debt. Investigates each category from `rules/tech-debt-checklist.md` using Next.js-specific procedures, plus framework-specific checks, by launching one `tech-debt-auditor` subagent per category in parallel. Outputs a prioritized report with file paths, line numbers, and remediation suggestions.

## Steps

### Step 1: Project Detection

1. Verify the project is Next.js by checking for `next.config.*` and `next` in `package.json` dependencies
   - If not a Next.js project → display "Error: This does not appear to be a Next.js project." and exit
2. Confirm App Router usage by checking for an `app/` directory
   - If only `pages/` exists → display "Warning: This project uses Pages Router. This skill targets App Router." and exit
3. Scan the project structure to understand the layout:
   - List route segments under `app/`
   - Identify `src/` vs root-level organization
   - Note the presence of `lib/`, `components/`, `hooks/`, `utils/`, or similar directories
   - Count the total number of `.ts`/`.tsx` source files scanned (excluding `node_modules`, build output, and generated files) — this becomes `Files scanned` in the Step 3 report; the main thread computes this once here, since no subagent has visibility into the whole project.
4. Compose a concise **project structure summary** from the above — this is passed as project context to every subagent in Step 2. Keep it to a few lines, e.g.:

   ```text
   Project: <name from package.json> (Next.js App Router)
   Route segments: <list or count under app/>
   Layout: src/ | root-level
   Key directories present: lib/, components/, hooks/, utils/ (as found)
   Files scanned: <count of .ts/.tsx source files>
   ```

### Step 2: Parallel Category Audit

Investigate all 17 categories in parallel via the generic `tech-debt-auditor` subagent — 12 common categories from `rules/tech-debt-checklist.md` (audited in the Next.js context) plus 5 Next.js-specific categories. Each category is independent, so launch every subagent together.

#### Common Categories

##### Code Duplication

- Search for repeated JSX patterns across components (e.g., similar form layouts, card structures)
- Check for duplicated data-fetching logic across route segments
- Look for repeated utility functions across `lib/`, `utils/`, or `helpers/` directories

##### Architecture & Layering

- Check for database or ORM calls directly in components (should be in server actions or API routes)
- Verify `"use client"` components do not import server-only modules
- Look for circular imports via manual inspection of import chains between modules

##### Error Handling

- List route segments missing `error.tsx` boundaries
- Check API route handlers (`route.ts`) for missing try-catch or error responses
- Search for empty catch blocks or swallowed errors

##### Type Safety

- Search for `any` type usage across `.ts` and `.tsx` files
- Check component props for missing type definitions (inline `props: any` or untyped destructuring)
- Look for type assertions (`as`) that bypass proper typing

##### Dead Code

- Identify unused exports by grepping for exported symbols and checking whether they are imported anywhere else in the codebase
- Search for commented-out code blocks
- Check for unused imports

##### Constants & Configuration

- Search for magic numbers and hard-coded string literals (URLs, API endpoints, thresholds)
- Check for environment-specific values not using `process.env` or `next.config.*`
- Look for duplicated constant definitions across files

##### Component / Module Size

- Identify `.tsx` files exceeding 300 lines
- Flag functions or components exceeding 100 lines
- Check for deeply nested JSX (4+ levels of nesting)

##### Dependency Management

- Check `package.json` for potentially unused dependencies
- Look for duplicate packages providing the same functionality (e.g., multiple date libraries)
- Note any dependency whose name or pinned version is a well-known deprecated/discontinued package (do not run install or audit commands)

##### Testing

- Compare route/component count against test file count to estimate coverage gaps
- Check for the presence of test configuration (`jest.config.*`, `vitest.config.*`, or similar)
- Identify critical paths (authentication, payment, data mutations) lacking tests

##### Accessibility

- Search for `<img>` tags missing `alt` attributes (should use `next/image` with `alt`)
- Check interactive elements (`<button>`, `<a>`, `<input>`) for missing `aria-*` labels
- Look for click handlers on non-interactive elements (`<div onClick>`)

##### Performance

- Check for unnecessary `"use client"` directives on components that could be Server Components
- Look for `<img>` tags instead of `next/image` (missing automatic optimization)
- Search for `<a>` tags instead of `next/link` (missing prefetching)
- Identify large client-side bundles by checking `"use client"` files that import heavy libraries

##### Security

- Check for `dangerouslySetInnerHTML` usage without sanitization
- Look for exposed secrets — `NEXT_PUBLIC_` env vars that should be server-only
- Verify API route handlers validate and sanitize input

#### Next.js-Specific Categories

##### Metadata

- List `page.tsx` and `layout.tsx` files missing `metadata` or `generateMetadata` exports
- Flag pages that lack Open Graph or description metadata

##### Error and Loading Boundaries

- List route segments missing `error.tsx` (especially segments with data fetching)
- List route segments missing `loading.tsx` or Suspense boundaries for async operations
- Check that `error.tsx` files include `"use client"` directive and a reset mechanism

##### Client vs Server Component Balance

- List all `"use client"` files and assess whether each truly requires client-side interactivity
- Flag `"use client"` components that only render static content or pass data through
- Check for large component trees pulled into the client bundle unnecessarily

##### Props Drilling

- Identify components receiving 5+ props that could indicate drilling
- Check for prop chains spanning 3+ component levels
- Suggest React Context or custom hooks where patterns indicate shared state

##### Suspense Boundaries

- Check async Server Components for missing Suspense wrappers
- Identify `fetch` calls in components without loading states
- Verify streaming patterns are used for slow data sources

#### Launching the Subagents

Issue **all 17 `Agent` calls in a single message** so they run concurrently, mirroring `review-team-run`'s precedent. Set `subagent_type` to `tech-debt-auditor` for every call. Each call's prompt must supply:

1. **Category** — the category name, matching the `#####` heading above **exactly** (e.g. `Code Duplication`, `Component / Module Size`, `Error and Loading Boundaries`). This exact string is returned in each finding's `category` field and is the join key Step 3 uses to populate the Summary table — do not paraphrase or abbreviate it.
2. **Investigation instructions** — that category's exact bullets from the lists above, verbatim.
3. **Project context** — the project structure summary composed in Step 1.

Wait for all 17 to return before continuing. Collect each subagent's finding-blocks (or `No findings.`), keyed by category, for Step 3.

### Step 3: Generate Report

Aggregate the 17 subagents' returned finding-blocks into a single report:

1. **Verify completion**: confirm all 17 subagent calls returned. For any call that errored, timed out, or returned output that is neither `No findings.` nor a well-formed finding-block, do not treat it as "no findings" — mark that category's Summary table row as `incomplete` instead of `-`/counts.
2. Parse every subagent's finding-blocks; subagents that returned `No findings.` contribute nothing.
3. **De-duplicate**: when two or more subagents report the same `file:line` with the same substance (this can happen since a few categories' instructions overlap, e.g. `Error Handling` and `Error and Loading Boundaries`), merge into one entry and list all contributing categories, e.g. `**[Error Handling, Error and Loading Boundaries]**`.
4. Bucket findings by `priority` (HIGH/MEDIUM/LOW) using the priority each subagent already assigned — do not re-derive it.
5. Assign **one continuous global number 1..N across all three buckets combined** (not per-category), ordered HIGH → MEDIUM → LOW.
6. Map each finding's `category` and `recommendation` fields onto the template's `**[<Category>]**` and `**Recommendation:**` fields.
7. Confirm the Summary table's `**Total**` row sums to N; if it does not, re-check steps 2–5 before presenting the report.

Output the audit results in the following format:

```text
## Technical Debt Audit Report — Next.js

Project: <project name from package.json>
Scan date: <YYYY-MM-DD>
Files scanned: <count>

### HIGH Priority (<count> items)

1. **[<Category>]** `path/to/file.ts:L<line>`
   **Finding:** <description of the issue>
   **Recommendation:** <how to fix>

### MEDIUM Priority (<count> items)
...

### LOW Priority (<count> items)
...

### Summary

| Category | HIGH | MEDIUM | LOW |
|---|---|---|---|
| Code Duplication | - | - | - |
| Architecture & Layering | - | - | - |
| Error Handling | - | - | - |
| Type Safety | - | - | - |
| Dead Code | - | - | - |
| Constants & Configuration | - | - | - |
| Component / Module Size | - | - | - |
| Dependency Management | - | - | - |
| Testing | - | - | - |
| Accessibility | - | - | - |
| Performance | - | - | - |
| Security | - | - | - |
| Metadata | - | - | - |
| Error and Loading Boundaries | - | - | - |
| Client vs Server Component Balance | - | - | - |
| Props Drilling | - | - | - |
| Suspense Boundaries | - | - | - |
| **Total** | **X** | **X** | **X** |
```

#### Priority Criteria

Priorities are assigned by the `tech-debt-auditor` subagent at investigation time (Step 3 does not re-derive them) — see that agent's Priority Criteria table for the HIGH/MEDIUM/LOW definitions.

### Step 4: Issue Creation

After presenting the report, ask the user which findings they want to create as GitHub Issues:

1. Display a numbered list of all findings (the globally-numbered list from Step 3) and prompt:

   ```text
   Issue化する項目を番号で指定してください（例: 1,3,5 / all / none）
   ```

2. Based on the user's response:
   - **`none`**: End the skill
   - **`all`**: Create an Issue for every finding
   - **Specific numbers**: Create Issues for selected findings only
3. For each selected finding, create an Issue using `/git-issue-create` conventions:
   - **Title**: Japanese, concise description of the finding
   - **Labels**: `enhancement` + priority label mapped from the report (`HIGH` → `priority: high`, `MEDIUM` → `priority: medium`, `LOW` → `priority: low`)
   - **Body**: Include the category, file path, finding details, and recommendation
4. Display the list of created Issues with their numbers and URLs
