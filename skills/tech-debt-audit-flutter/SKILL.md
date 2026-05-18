---
name: tech-debt-audit-flutter
description: Audit technical debt in Flutter / Dart projects - analyze code against checklist and generate prioritized report
---

# Flutter Technical Debt Audit Skill

Audit a Flutter / Dart project for technical debt. Investigates each category from `rules/tech-debt-checklist.md` using Flutter / Dart-specific procedures, plus framework-specific checks (Widget bloat, state management consistency, generated code freshness, etc.). Outputs a prioritized report with file paths, line numbers, and remediation suggestions.

## Steps

### Step 1: Project Detection

1. Verify the project is a Flutter / Dart project by checking for `pubspec.yaml` at the repository root and confirming it contains a `flutter:` block or a `flutter` SDK constraint under `dependencies:`
   - If `pubspec.yaml` is absent → display "Error: This does not appear to be a Dart project (no pubspec.yaml found)." and exit
   - If `pubspec.yaml` exists but has no `flutter` dependency → display "Warning: This appears to be a pure Dart package. Flutter-specific checks (Step 3 Widget / State / Navigation) will be skipped." and continue
2. Confirm the application entry point exists at `lib/main.dart` (for application projects). Library packages without `main.dart` are still valid audit targets
3. Determine the Flutter CLI prefix to reference in subsequent commands:
   - If `.fvmrc` or `.fvm/` exists → use `fvm flutter ...` / `fvm dart ...`
   - Otherwise → use `flutter ...` / `dart ...`
4. Scan the project structure to understand the layout:
   - List top-level directories under `lib/` (e.g., `app/`, `core/`, `features/`)
   - For each feature directory, note whether a clean-architecture layering (`presentation/` / `application/` / `domain/` / `data/`) is used, or some other structure
   - Note the presence and shape of `test/`, `integration_test/`, `analysis_options.yaml`, `.github/workflows/`

### Step 2: Common Checklist Audit

Investigate each category from `rules/tech-debt-checklist.md` in the Flutter / Dart context:

#### Code Duplication

- Search for repeated Widget tree patterns across screens (e.g., similar `Scaffold` + `AppBar` + `ListView` skeletons, repeated `Padding` / `Container` nesting)
- Check for duplicated data-fetching or Repository implementations across features
- Look for repeated utility functions across `lib/core/` and `features/*/`
- Flag duplicated `freezed` model fields that suggest a missing shared value object

#### Architecture & Layering

- If the project uses feature-first / clean-architecture layering, check for cross-layer violations:
  - `presentation/` files importing from `data/` directly (should go through `application/`)
  - `domain/` files importing Flutter, Riverpod, Drift, or any framework
  - `application/` files importing widgets from `presentation/`
- Look for direct cross-feature imports (`features/foo/` importing `features/bar/`) — shared code belongs in `core/`
- Detect circular import chains (manual grep on suspected cycles, or `dart run import_sorter` / `dart run dependency_validator` if configured)
- Check whether stateful logic that belongs in `application/` (Riverpod notifiers) is leaking into `StatefulWidget` setState blocks

#### Error Handling

- Search for `await` calls outside any `try / catch` where the awaited future can fail (I/O, network, plugin calls)
- Look for empty `catch` blocks (`catch (_) {}`) and overly broad `catch (e)` that swallow stack traces
- Check `Future` / `Stream` consumers for missing `onError` and for `unawaited()` calls that ignore failures
- In the UI layer, check `AsyncValue` consumers (`ref.watch(...)`) for missing `error` branches (`when` / `whenData` that drops errors)
- Search for `throw Exception('...')` strings that should be typed error classes

#### Type Safety

- Search `.dart` files for `dynamic` parameter / return types and `Object?` used as a bag of values
- Flag null-assertion operator (`!`) usage that bypasses null safety
- Search for `as` casts that could fail at runtime
- Look for `Map<String, dynamic>` passed through multiple layers instead of being parsed into a `freezed` model at the boundary
- Flag missing type annotations on `var` declarations whose initializer type is non-obvious

#### Dead Code

- Run `<flutter-prefix> analyze` and collect `unused_element`, `unused_import`, `unused_local_variable`, `dead_code` diagnostics
- Search for commented-out code blocks (3+ consecutive comment lines that look like code)
- Check for `export` statements that are never re-imported
- Identify TODO / FIXME / XXX comments older than the current sprint that may indicate abandoned work

#### Constants & Configuration

- Search for magic numbers in `build()` methods (raw padding / radius / duration literals that should be theme tokens or named constants)
- Look for hard-coded URLs, API base paths, or feature-flag strings embedded in source files
- Check for `const` opportunities missed on `Widget` constructors (analyzer rule `prefer_const_constructors`)
- Identify duplicated `Duration` / `EdgeInsets` / color literal definitions across files

#### Component / Module Size

- Identify `.dart` files exceeding **300 lines** (excluding generated `*.g.dart` / `*.freezed.dart`)
- Flag `build()` methods exceeding **100 lines** — candidates for `extract widget` refactor
- Flag any single Widget tree exceeding **4 levels of nesting**
- Flag classes with more than ~10 public methods that mix unrelated concerns

#### Dependency Management

- Cross-reference `pubspec.yaml` dependencies against actual imports in `lib/` to find unused packages
- Run `<flutter-prefix> pub outdated` and flag packages multiple major versions behind
- Look for duplicate packages providing overlapping functionality (e.g., two HTTP clients, two date libraries, two mocking frameworks)
- Flag git-ref or path dependencies (`git:` / `path:` entries) that bypass version pinning
- Check for `dev_dependencies` entries that are not actually referenced in `test/`, `build_runner` configs, or CI

#### Testing

- Run `<flutter-prefix> test --coverage` (or note its absence) and inspect `coverage/lcov.info` for uncovered files
- Compare the file count under `lib/features/*/` against `test/features/*/` to estimate coverage gaps per feature
- Identify critical paths (auth, payment, persistence, sync) lacking widget or integration tests
- Look for tests that mock the Repository instead of using an in-memory implementation, which may hide integration bugs
- Flag tests that depend on real `DateTime.now()`, `Random()`, or file-system state without injection

#### Accessibility

- Search for `GestureDetector` / `InkWell` wrapping non-interactive content without a `Semantics` label
- Check `Image`, `Icon`, and `CircleAvatar` for missing `semanticLabel` / `semanticsLabel`
- Look for `IconButton` / `FloatingActionButton` without `tooltip`
- Flag color-only signals (red / green text without an accompanying icon or label)
- Check `TextField` for missing `labelText` / `hintText` semantics

#### Performance

- Look for expensive computations inside `build()` (sorting, JSON parsing, regex compilation) that should be memoized in `application/`
- Flag `ListView(...)` with a large hard-coded children list — should be `ListView.builder` with `itemExtent` when feasible
- Search for `setState` calls in hot paths (animation tick, scroll listeners) that rebuild large subtrees
- Identify `StreamSubscription`, `TextEditingController`, `AnimationController`, `FocusNode` that are created but never disposed
- Look for missing `const` on widget constructors that disables widget-tree diffing optimization

#### Security

- Check `pubspec.yaml` for git / URL / path dependencies that pull from unvetted sources
- Search for secrets, API keys, or tokens embedded in source (`String _apiKey = '...'`)
- Flag use of `SharedPreferences` for sensitive data that should live in `flutter_secure_storage` / Keychain
- Look for unvalidated external input passed directly into SQL, file paths, or platform channels
- Check for `http://` (non-TLS) URLs hard-coded in network calls

### Step 3: Flutter / Dart-Specific Audit

Checks specific to Flutter / Dart conventions that are not covered by the common checklist.

#### Widget / build Method Bloat

- List `build()` methods exceeding 100 lines
- Identify Widget trees nested 4+ levels deep without intermediate `extract widget` refactors
- Flag screens that mix layout, data transformation, and event handling in a single `build()`

#### State Management Consistency

- Detect mixing of state-management approaches (`setState` + Riverpod + Provider + Bloc in the same project)
- If the project uses Riverpod, flag hand-written `Provider(...)` / `StateProvider(...)` that should be `@riverpod` generated providers
- If the project uses Riverpod, flag `StatefulWidget` instances holding business state that should live in a notifier
- Look for global mutable singletons (`final foo = Foo();` at top-level) that bypass the state-management layer

#### Async / Stream Handling

- Identify `FutureBuilder` / `StreamBuilder` usage in projects where Riverpod `AsyncValue` would be more consistent
- Flag uncancelled `StreamSubscription` (created in `initState` without `cancel()` in `dispose`)
- Flag undisposed `TextEditingController` / `AnimationController` / `FocusNode` / `ScrollController`
- Check `Timer` and `Future.delayed` usage in widgets for missing cancellation on dispose

#### Generated Code Freshness

- Run `<flutter-prefix> dart run build_runner build --delete-conflicting-outputs` and check `git status`
- If any `*.g.dart` or `*.freezed.dart` files changed → the committed generated code is stale and must be regenerated
- Verify that generated files are committed to the repo (per project convention) and not ignored

#### Navigation Conventions

- If the project uses `go_router`, search for direct `Navigator.push` / `Navigator.pop` / `Navigator.of(context)` usage that should go through `context.push` / `context.go` / `context.pop`
- Check that route definitions are centralized (e.g., `lib/app/router.dart`) and not scattered across screens
- Flag deep-link parameters parsed manually inside screens instead of declared on the route

#### Layer Boundary Audit (project-specific)

- If the project's `CLAUDE.md` or `.claude/rules/architecture.md` defines layering rules, verify each `lib/features/<feature>/` directory matches the prescribed structure
- Flag features missing a layer (e.g., `presentation/` only with no `application/`) where the convention requires the full set
- Check that `core/` is not used as a junk-drawer — utilities used by a single feature should live in that feature

#### Build / CI Coverage Gaps

- Inspect `.github/workflows/*.yml` for which checks run on PRs
- Flag missing CI steps that the audit considers important:
  - `<flutter-prefix> format --output=none --set-exit-code .` (or `dart format`)
  - `<flutter-prefix> test --coverage` with a coverage reporter
  - `<flutter-prefix> pub outdated --exit-if-no-update-needed=false` as a non-blocking advisory
  - Generated-code freshness check (`build_runner build` + `git diff --exit-code`)
- Note these as LOW priority unless the project has had recent regressions from the missing check

### Step 4: Generate Report

Output the audit results in the following format:

```text
## Technical Debt Audit Report — Flutter

Project: <project name from pubspec.yaml>
Scan date: <YYYY-MM-DD>
Files scanned: <count>

### HIGH Priority (<count> items)

1. **[<Category>]** `path/to/file.dart:L<line>`
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
| Widget / build Bloat | - | - | - |
| State Management Consistency | - | - | - |
| Async / Stream Handling | - | - | - |
| Generated Code Freshness | - | - | - |
| Navigation Conventions | - | - | - |
| Layer Boundary | - | - | - |
| Build / CI Coverage Gaps | - | - | - |
| **Total** | **X** | **X** | **X** |
```

#### Priority Criteria

| Priority | Criteria |
|---|---|
| **HIGH** | Security risks, potential data loss, production failures, undisposed resources causing leaks, unhandled errors on critical paths |
| **MEDIUM** | Significant maintainability or readability degradation, performance impact, missing type safety, state-management inconsistency |
| **LOW** | Best-practice deviations, code quality improvements, cosmetic issues, advisory CI gaps |

### Step 5: Issue Creation

After presenting the report, ask the user which findings they want to create as GitHub Issues:

1. Display a numbered list of all findings and prompt:

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
