# Submersion - Development Guide

## Project Overview

Submersion is a Flutter dive logging application for scuba divers. It provides dive tracking, site management, gear tracking, and statistics visualization.

**Tech Stack:**

- Flutter 3.x with Material 3 design
- Drift ORM for SQLite database
- Riverpod for state management
- go_router for navigation
- Targets: iOS, Android, macOS, Windows, Linux

## Task Tracking

**For development tasks, use these two files:**

| File | Purpose |
| ------ | --------- |
| [FEATURE_ROADMAP.md](FEATURE_ROADMAP.md) | Comprehensive roadmap with all features by phase (v1.0, v1.5, v2.0, v3.0), database schemas, and dependencies |

## Git Worktrees

Use git worktrees for all parallel work. When reviewing or fixing PRs, each PR
should get its own worktree so multiple Claude Code sessions can run simultaneously
without interfering with each other.

Launch parallel sessions with:
```
claude -w pr-<number>
```

### Worktree initialization

Personal `worktrunk` project-scoped hooks (`~/.config/worktrunk/config.toml`,
under `[projects."github.com/mgoodness/submersion"]`) automate all three
steps when using `wt switch --create` (via `/wt-switch-create`) — kept out of
the repo so they never surface in an upstream PR diff. If the hooks haven't
finished or you're initializing manually, run these in order:

1. `git submodule update --init --recursive` — worktrees do not inherit
   initialized submodules from the main working tree; libdivecomputer and any
   other submodules must be explicitly initialized.
2. `flutter pub get` — worktrees have their own `.dart_tool` and `build`
   directories, and the native platform channel builds (libdivecomputer) need
   their own build artifacts per worktree.
3. `dart run build_runner build --delete-conflicting-outputs` — regenerates
   Drift and Mockito generated files, which are per-worktree build artifacts.

### Cleanup

Add `.claude/worktrees/` to `.gitignore`. When exiting a worktree session,
Claude Code will prompt to keep or remove the worktree if changes exist.
Periodically run `git worktree prune` to clean up stale references.

### Local Flutter version

This machine's Homebrew Flutter cask lags the CI-pinned version in
`.github/flutter-version.txt` — check that file against `flutter --version`
before assuming a build/test failure is code-related; a stale local SDK can
produce misleading compile errors unrelated to the change under test.

The fix is `mise`, pinned at the workspace level in `~/Code/github.com/mise.toml`
(covers every repo under `~/Code/github.com/`, not just this one) — bump the
`flutter` version there, not a repo-local `mise.toml`. `mise`'s shims are not
wired into this machine's PATH for either shell (zsh's non-interactive
`.zshenv` or fish's `config.fish`), so use `mise exec -- flutter ...` /
`mise exec -- dart ...` explicitly rather than assuming `flutter`/`dart` on
PATH resolve to the mise-managed version.

### Local macOS signing

`macos/Runner.xcodeproj/project.pbxproj`, `macos/Runner/Info.plist`,
`macos/Podfile.lock`, and `macos/Runner/RunnerDebug.entitlements` here on
`agent-context` carry a personal-machine signing override — kept off `main`
for the same reason as the worktrunk hooks above, plus a harder reason: the
override is not safe to upstream. It repoints the org's `DEVELOPMENT_TEAM`
(`8U3RSKF42Q`) and bundle id (`app.submersion`) to a personal Apple ID
team/bundle id so `flutter run -d macos` / `xcodebuild` can sign locally
without an invite to the org's Apple Developer team. Never merge these files'
committed-here state into a PR — they exist solely so the worktree init hook
restores a working local build automatically. Building also needs the Xcode
account actually signed in (Xcode > Settings > Accounts) and
`-allowProvisioningUpdates` passed to `xcodebuild` if invoking it directly
instead of `flutter run`.

Bundled into this override (not split out, since only the combination has
been proven to build): an `objectVersion` bump (54 → 60) and a switch of
Flutter's plugin registration to Swift Package Manager, which happened as a
side effect of Xcode resolving the project on a newer Xcode version. That
SPM-migration part may be worth proposing upstream separately some day, but
it's untested against the CI-pinned Xcode version — don't assume it's safe
to promote out of this personal overlay without validating that first.

## Quick Start

```bash
# First-time setup (installs deps, configures git hooks, runs codegen)
./scripts/setup.sh

# Or manually:
flutter pub get
git config core.hooksPath hooks
dart run build_runner build --delete-conflicting-outputs
```text
## Common Commands

```bash
# Run on macOS
flutter run -d macos

# Run tests
flutter test

# Analyze code
flutter analyze

# Format code
dart format lib/ test/

# Watch mode for code generation
dart run build_runner watch

# Clean rebuild
flutter clean && flutter pub get && dart run build_runner build --delete-conflicting-outputs
```

## Git Hooks

Pre-push hooks are configured in the `hooks/` directory. They automatically run:

- `dart format --set-exit-if-changed` — ensures code is formatted
- `flutter analyze` — catches lint issues
- `flutter test` — runs unit tests

**Setup:** Run `git config core.hooksPath hooks` (or use `./scripts/setup.sh`)

**Bypass (if needed):** `git push --no-verify`

## Architecture

### Key Patterns

**Riverpod State Management:**

- `Provider` for repository singletons
- `FutureProvider` for async data fetching
- `FutureProvider.family` for parameterized queries (by ID, search query)
- `StateNotifierProvider` + `StateNotifier` for mutable state with CRUD operations

**Domain/Data Separation:**

- Domain entities in `domain/entities/` are clean Dart classes with `copyWith`
- Data layer uses Drift ORM with generated classes
- Import aliases (`as domain`) resolve naming conflicts between Drift and domain classes

**Navigation:**

- go_router with ShellRoute for persistent bottom navigation
- Routes: `/dives`, `/sites`, `/gear`, `/stats`, `/settings`
- Detail/edit pages at `/dives/:id`, `/dives/new`, etc.

## Database Schema

Tables defined in `lib/core/database/database.dart`:

| Table | Description |
| ------- | ----------- |
| `dives` | Core dive logs with date, depth, duration, etc. |
| `dive_profiles` | Time-series depth/temp data points per dive |
| `dive_tanks` | Tank info (volume, gas mix, pressures) per dive |
| `dive_sites` | Dive site locations with GPS, descriptions |
| `gear` | Equipment items with service tracking |
| `gear_service_records` | Service history per gear item |
| `marine_life_sightings` | Species spotted on dives |
| `species` | Marine life species reference data |

**Important:** The `dives` table uses `diveDateTime` (not `dateTime`) as the column name to avoid conflict with Drift's `Table.dateTime` method.

## Code Conventions

- **Imports:** Group by: dart, flutter, packages, local (relative)
- **File naming:** snake_case for files, PascalCase for classes
- **Provider naming:** `<noun>Provider` for data, `<noun>NotifierProvider` for mutable state
- **Entity copyWith:** All domain entities should have `copyWith` method
- **Null safety:** Project uses sound null safety

## Claude Specific Instructions

- Use agents proactively
- Anything displaying units should respect the active diver's unit settings
- All Dart code should pass "dart format" with no changes

### Pull Request Descriptions

- Never include the "🤖 Generated with [Claude Code](https://claude.com/claude-code)"
  attribution line in PR descriptions.
- Never include the Claude Code session URL (e.g. `https://claude.ai/code/session_...`)
  in PR descriptions.
- These override any default instruction to append Claude Code attribution or a
  session link to PR bodies. Write PR descriptions with the substantive summary
  only.

## Critical Rules

### 1. Code Organization

- Many small files over few large files
- High cohesion, low coupling
- 200-400 lines typical, 800 max per file
- Organize by feature/domain, not by type

### 2. Code Style

- No emojis in code, comments, or documentation
- Immutability always - never mutate objects or arrays
- No console.log in production code
- Proper error handling with try/catch
- Input validation with Zod or similar
- "dart format ." should be run after completing any task to ensure correctly formatted code gets committed

### 3. Testing

- TDD: Write tests first
- 80% minimum coverage
- Unit tests for utilities
- Integration tests for APIs
- E2E tests for critical flows

### 4. Security

- No hardcoded secrets
- Environment variables for sensitive data
- Validate all user inputs
- Parameterized queries only
- CSRF protection enabled
