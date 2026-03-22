---
name: tech-debt-audit-nextjs
description: Audit technical debt in Next.js (App Router) projects - analyze code against checklist and generate prioritized report
---

# Next.js Technical Debt Audit Skill

Audit a Next.js (App Router) project for technical debt. Investigates each category from `rules/tech-debt-checklist.md` using Next.js-specific procedures, plus framework-specific checks. Outputs a prioritized report with file paths, line numbers, and remediation suggestions.

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

### Step 2: Common Checklist Audit

Investigate each category from `rules/tech-debt-checklist.md` in the Next.js context:

#### Code Duplication
- Search for repeated JSX patterns across components (e.g., similar form layouts, card structures)
- Check for duplicated data-fetching logic across route segments
- Look for repeated utility functions across `lib/`, `utils/`, or `helpers/` directories

#### Architecture & Layering
- Check for database or ORM calls directly in components (should be in server actions or API routes)
- Verify `"use client"` components do not import server-only modules
- Look for circular imports with `madge` or manual inspection of import chains

#### Error Handling
- List route segments missing `error.tsx` boundaries
- Check API route handlers (`route.ts`) for missing try-catch or error responses
- Search for empty catch blocks or swallowed errors

#### Type Safety
- Search for `any` type usage across `.ts` and `.tsx` files
- Check component props for missing type definitions (inline `props: any` or untyped destructuring)
- Look for type assertions (`as`) that bypass proper typing

#### Dead Code
- Identify unused exports with TypeScript or ESLint reports if available
- Search for commented-out code blocks
- Check for unused imports

#### Constants & Configuration
- Search for magic numbers and hard-coded string literals (URLs, API endpoints, thresholds)
- Check for environment-specific values not using `process.env` or `next.config.*`
- Look for duplicated constant definitions across files

#### Component / Module Size
- Identify `.tsx` files exceeding 300 lines
- Flag functions or components exceeding 100 lines
- Check for deeply nested JSX (4+ levels of nesting)

#### Dependency Management
- Check `package.json` for potentially unused dependencies
- Look for duplicate packages providing the same functionality (e.g., multiple date libraries)
- Note any dependencies with known deprecation warnings

#### Testing
- Compare route/component count against test file count to estimate coverage gaps
- Check for the presence of test configuration (`jest.config.*`, `vitest.config.*`, or similar)
- Identify critical paths (authentication, payment, data mutations) lacking tests

#### Accessibility
- Search for `<img>` tags missing `alt` attributes (should use `next/image` with `alt`)
- Check interactive elements (`<button>`, `<a>`, `<input>`) for missing `aria-*` labels
- Look for click handlers on non-interactive elements (`<div onClick>`)

#### Performance
- Check for unnecessary `"use client"` directives on components that could be Server Components
- Look for `<img>` tags instead of `next/image` (missing automatic optimization)
- Search for `<a>` tags instead of `next/link` (missing prefetching)
- Identify large client-side bundles by checking `"use client"` files that import heavy libraries

#### Security
- Check for `dangerouslySetInnerHTML` usage without sanitization
- Look for exposed secrets — `NEXT_PUBLIC_` env vars that should be server-only
- Verify API route handlers validate and sanitize input

### Step 3: Next.js-Specific Audit

Checks specific to Next.js App Router conventions:

#### Metadata
- List `page.tsx` and `layout.tsx` files missing `metadata` or `generateMetadata` exports
- Flag pages that lack Open Graph or description metadata

#### Error and Loading Boundaries
- List route segments missing `error.tsx` (especially segments with data fetching)
- List route segments missing `loading.tsx` or Suspense boundaries for async operations
- Check that `error.tsx` files include `"use client"` directive and a reset mechanism

#### Client vs Server Component Balance
- List all `"use client"` files and assess whether each truly requires client-side interactivity
- Flag `"use client"` components that only render static content or pass data through
- Check for large component trees pulled into the client bundle unnecessarily

#### Props Drilling
- Identify components receiving 5+ props that could indicate drilling
- Check for prop chains spanning 3+ component levels
- Suggest React Context or custom hooks where patterns indicate shared state

#### Suspense Boundaries
- Check async Server Components for missing Suspense wrappers
- Identify `fetch` calls in components without loading states
- Verify streaming patterns are used for slow data sources

### Step 4: Generate Report

Output the audit results in the following format:

```
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
| Component Size | - | - | - |
| Dependency Management | - | - | - |
| Testing | - | - | - |
| Accessibility | - | - | - |
| Performance | - | - | - |
| Security | - | - | - |
| Metadata | - | - | - |
| Error/Loading Boundaries | - | - | - |
| Client vs Server Balance | - | - | - |
| Props Drilling | - | - | - |
| Suspense Boundaries | - | - | - |
| **Total** | **X** | **X** | **X** |
```

#### Priority Criteria

| Priority | Criteria |
|---|---|
| **HIGH** | Security risks, potential data loss, production failures, missing error boundaries on critical paths |
| **MEDIUM** | Significant maintainability or readability degradation, performance impact, missing type safety |
| **LOW** | Best-practice deviations, code quality improvements, cosmetic issues |

### Step 5: Issue Creation

After presenting the report, ask the user which findings they want to create as GitHub Issues:

1. Display a numbered list of all findings and prompt:
   ```
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
