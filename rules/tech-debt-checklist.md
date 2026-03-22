# Technical Debt Checklist

A language- and framework-agnostic list of technical debt perspectives. Language-specific audit skills (e.g., `tech-debt-audit-nextjs`) reference this checklist as their common foundation.

This file lists **what to look for**, not how to investigate. Investigation procedures are defined in each audit skill.

## Code Duplication

- Repeated logic that could be extracted into a shared function or module
- Copy-pasted code blocks with only minor parameter differences
- Multiple implementations of the same algorithm or business rule
- Duplicated constants, configuration values, or validation rules across files

## Architecture & Layering

- Layer violations (e.g., UI layer directly accessing the database)
- Circular dependencies between modules or packages
- Mixed responsibilities within a single module (e.g., business logic in a controller)
- Tight coupling between components that should be independent
- Missing or inconsistent abstraction boundaries

## Error Handling

- Missing error handling for operations that can fail (I/O, network, parsing)
- Swallowed errors (empty catch blocks, ignored return values)
- Overly broad catch blocks that mask specific failure modes
- Inconsistent error reporting patterns across the codebase
- Missing user-facing error messages or fallback behavior

## Type Safety

- Use of `any`, `unknown`, or equivalent escape-hatch types without justification
- Missing type definitions for function parameters, return values, or data structures
- Insufficient type guards or runtime type checks at system boundaries
- Type assertions (`as`) used to bypass compiler checks instead of fixing the underlying type

## Dead Code

- Unused functions, classes, methods, or variables
- Unused imports or dependencies
- Commented-out code left in the codebase
- Feature flags or conditional branches that are no longer reachable
- Unreachable code paths after early returns or throws

## Constants & Configuration

- Magic numbers or string literals embedded in logic
- Inconsistent definitions of the same constant across files
- Hard-coded values that should be configurable (URLs, thresholds, limits)
- Environment-specific values not externalized to configuration

## Component / Module Size

- Functions or methods exceeding a reasonable line count for their complexity
- Classes or modules with too many responsibilities (God objects)
- Files that mix unrelated concerns
- Deeply nested control flow that reduces readability

## Dependency Management

- Unused dependencies listed in the manifest (package.json, pom.xml, etc.)
- Outdated dependencies with known vulnerabilities or breaking changes
- Duplicate packages providing the same functionality
- Pinned versions that prevent security patches

## Testing

- Critical paths without test coverage
- Brittle tests that break on unrelated changes
- Tests that depend on execution order or shared mutable state
- Missing edge-case or boundary-value tests
- Test suites that are too slow to run frequently

## Accessibility

- Interactive elements missing keyboard support
- Missing or inadequate alternative text, labels, or ARIA attributes
- Insufficient color contrast or reliance on color alone to convey information
- Missing focus management for dynamic content changes

## Performance

- Inefficient algorithms where better alternatives exist (e.g., O(n^2) vs O(n log n))
- Unnecessary re-computation or re-rendering of unchanged data
- N+1 query patterns in data access layers
- Missing pagination or unbounded data fetching
- Resource leaks (unclosed connections, streams, or handles)

## Security (Technical Debt Perspective)

Items here focus on debt-like security gaps. For secret-management rules, see `security.md`.

- Missing input validation or sanitization at system boundaries
- Use of deprecated or insecure cryptographic functions
- Overly permissive access controls or default-allow patterns
- Missing rate limiting or abuse prevention on public endpoints
- Sensitive data logged or exposed in error responses
