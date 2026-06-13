# Add Cavern dive type

Status: ready-for-human
PR: #328
Category: enhancement
Reporter: ericgriffin
Source: issue #314

## Agent Brief

**Category:** enhancement
**Summary:** Add "Cavern" as a built-in dive type alongside the existing "Cave" type.

**Current behavior:**
The app ships with a fixed set of built-in dive types seeded at database creation.
"Cave" is present. "Cavern" is not. Users who want to log cavern dives must either
use "Cave" (inaccurate) or create a custom dive type (not discoverable by default).

**Desired behavior:**
"Cavern" appears as a built-in dive type in the dive type picker alongside all other
built-in types. It is distinct from "Cave": cavern diving stays within the daylight
zone near a cave entrance and requires only cavern certification, whereas cave diving
ventures beyond the light zone and requires full cave training. Both are standard
PADI/NAUI/SSI/TDI categories.

The new type must be available on fresh installs and on existing databases that
upgrade to the new schema version.

**Key interfaces:**
- The built-in dive types seed list — add `('cavern', 'Cavern', <sort_order>)` so
  it appears after "Liveaboard" in the default sort order
- The database schema version — bump by one and add a corresponding migration block
  that inserts the new type with `INSERT OR IGNORE` (so it is safe to run on any
  existing database regardless of whether the user already created a custom type
  named "cavern")
- The migration version registry — append the new schema version number so the
  progress step counter stays accurate
- No changes to the `DiveType` entity, repository, or UI — the existing
  infrastructure handles any built-in type automatically

**Acceptance criteria:**
- [x] A fresh install shows "Cavern" in the dive type list
- [x] An existing database that upgrades shows "Cavern" in the dive type list
  after migration completes
- [x] "Cavern" and "Cave" are both present and independently selectable
- [x] A database that already has a user-created type with id "cavern" is not
  affected (INSERT OR IGNORE prevents a conflict)
- [x] `flutter test` passes with no changes to existing tests

**Out of scope:**
- Localization of the "Cavern" name (dive type names are stored as plain strings
  in the database, not as localization keys)
- Reordering existing built-in types
- Adding any other new dive types
- UI changes to the dive type picker or detail view

## Comments

*2026-06-12 — triage by Claude:* Cavern is a recognized diving discipline distinct
from Cave. Implementation is a one-line seed addition plus a migration block;
all UI and repository infrastructure already handles any built-in type.
Moved to ready-for-agent.

*2026-06-12 — implemented by Claude:* schema v77, INSERT OR IGNORE migration with
sqlite_master guard. 3 TDD cycles (fresh install, upgrade, idempotency).
8045 tests passing. PR #328 open.
