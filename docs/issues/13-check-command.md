# Title

`check` command with console and JSON reports

## Summary

Implement `i18n-agent check`: load config → parse catalogs → diff → render console/JSON
report → exit 0 (clean) or 1 (pending work). Establish the shared report renderers used
by all later commands.

## Context

`check` is the read-only CI gate (DESIGN §3.3, §13) and the first end-to-end wiring of
config + adapters + diff. The `RunReport` JSON shape defined here (DESIGN §15.2) is a
stable consumer contract for every command.

## Scope

- In: `src/cli/commands/check.ts`, `src/report/{types,console,json}.ts`, file-set
  resolution helper (`config files[] → concrete source/target paths`), CLI flag parsing
  for `--locale`/`--file`, integration tests with fixture repos.
- Out: translation, lockfile writes (check never writes anything), PR body rendering
  (Issue 27/28).

## Detailed Requirements

1. Path resolution helper `resolveFilePaths(config, root)`: for each `files[]` entry,
   substitute `{locale}` (via `localeMap` when present) for source (or use `sourcePath`)
   and each target locale; return
   `{ fileId, format, sourcePath, targets: Map<locale,path> }[]`. Missing **source** file
   → `ConfigError` naming fileId+path. Missing target file → empty catalog (DESIGN §8.2).
2. Command flow: `loadConfig` → resolve paths → adapter.parse source+targets →
   `loadLockfile` (read-only) → `computeDiff` per file → aggregate `RunReport`
   (`command: "check"`, `outcome: "clean"|"pending"`).
3. Filters: `--locale <l>` (repeatable) restricts target locales (unknown locale →
   `UsageError` listing configured ones); `--file <id>` (repeatable) restricts file
   entries (unknown id → `UsageError`).
4. `report/types.ts`: implement the `RunReport` interface from DESIGN §15.2 verbatim,
   plus `renderConsole(report)` per §15.1 (per file×locale table with counts columns
   `missing stale adopted orphan translated failed`, totals row; plain ASCII table, no
   table dependency) and `renderJson(report)` (stable key order, `reportVersion: 1`).
5. Output discipline: report → stdout; logs → stderr (Issue 03). `--json` suppresses the
   console table.
6. Exit: `pending ∪ orphans ≠ ∅` → 1, else 0 (DESIGN §13). Config/format errors keep
   their own codes (3).
7. Integration tests (`tests/integration/check.test.ts`): fixture repo with json+yaml
   file groups × 2 locales covering all diff classes → assert exact JSON report (snapshot
   with stable fields only) and exit codes for: clean repo → 0; missing keys → 1;
   `--locale` filter changes result; corrupt config → 3; missing source file → 3.
8. Performance guard: 20 000-key generated fixture completes `check` in < 5 s in CI
   (coarse budget; catches accidental quadratic diff).

## Acceptance Criteria

- [ ] `check` never writes any file (test asserts fixture tree hash unchanged, lockfile
      absent stays absent).
- [ ] Exit codes: 0 clean / 1 pending / 3 config errors — all integration-tested.
- [ ] `--json` output validates against the `RunReport` type and is byte-stable across
      two runs on the same fixture.
- [ ] Console table renders all count columns and totals; respects `--quiet`.
- [ ] Filters behave per requirement 3 including error cases.

## Validation

- `npx vitest run tests/integration/check.test.ts tests/unit/report` green.
- Manual: run `check` (built CLI) against the fixture repo; verify table and `echo $?`.

## Dependencies

02, 03, 05, 06 (json adapter for first fixtures), 12

## Non-goals

Any mutation, provider calls, PR bodies, `validate` command (Issue 24), watch mode.

## Design References

- DESIGN §3.3, §13 (flags/exit codes), §15.1–15.2 (report shapes), §8.2 (missing targets)
