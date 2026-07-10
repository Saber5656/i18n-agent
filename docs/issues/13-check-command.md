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
   → `ConfigError` naming fileId+path. Missing target file → `emptyCatalog(fileId,
   locale)` from Issue 05 (`parse` is never called for absent files, DESIGN §8.2).
2. Command flow: `loadConfig` → resolve paths → adapter.parse source+targets →
   `loadLockfile` (read-only) → `computeDiff` per file → aggregate `RunReport`
   (`command: "check"`, `outcome: "clean"|"pending"`).
3. Filters: `--locale <l>` (repeatable) restricts target locales (unknown locale →
   `UsageError` listing configured ones); `--file <id>` (repeatable) restricts file
   entries (unknown id → `UsageError`).
4. `report/types.ts`: implement the `RunReport` interface from DESIGN §15.2 verbatim
   (incl. `pr`/`prs` fields), plus `renderConsole(report)` per §15.1 — per file×locale
   table with counts columns `missing stale adopted orphan translated failed warnings`,
   where `warnings` is derived as the count of `issues[]` entries with severity `warn`
   (no separate counter field); totals row; plain ASCII table, no table dependency —
   and `renderJson(report)` (stable key order, `reportVersion: 1`).
   `Catalog.warnings` (`AdapterWarning`, e.g. Android `plurals-skipped`) are mapped into
   `issues[]` as `{ validatorId: "format", severity: "warn", … }` by this command (and
   by later commands reusing the same helper `adapterWarningsToIssues`).
5. Output discipline: report → stdout; logs → stderr (Issue 03). `--json` suppresses the
   console table.
6. Exit: `pending ∪ orphans ≠ ∅` → 1, else 0 (DESIGN §13). Config/format errors keep
   their own codes (3).
7. Integration tests (`tests/integration/check.test.ts`): fixture repo with **two json
   file groups** × 2 locales covering all diff classes (other formats join the matrix in
   Issues 25/32) → assert exact JSON report (snapshot with stable fields only) and exit
   codes for: clean repo → 0; missing keys → 1; `--locale` filter changes result;
   corrupt config → 3; missing source file → 3.
8. Performance guard: 20 000-key generated fixture completes `check` in < 5 s in CI
   (coarse budget; catches accidental quadratic diff).

## Acceptance Criteria

- [ ] `check` never writes any file (test asserts fixture tree hash unchanged, lockfile
      absent stays absent).
- [ ] Exit codes: 0 clean / 1 pending / 3 config errors — all integration-tested.
- [ ] `--json` output: parsed object passes a compile-time `satisfies RunReport`
      assertion in the test, matches a committed snapshot, and is byte-stable across two
      runs on the same fixture.
- [ ] Console table renders all count columns (incl. derived `warnings`) and totals;
      respects `--quiet`.
- [ ] Filters behave per requirement 3 including error cases.
- [ ] Security posture note honored: path confinement and parser hardening are enforced
      by Issues 02/06–10 and re-audited in Issue 36; this command adds no new fs or
      parser surface beyond those modules (code-review checklist item in the PR).

## Validation

- `npx vitest run tests/integration/check.test.ts tests/unit/report` green.
- Manual: run `check` (built CLI) against the fixture repo; verify table and `echo $?`.

## Dependencies

02, 03, 05, 06 (json adapter for first fixtures), 12

## Non-goals

Any mutation, provider calls, PR bodies, `validate` command (Issue 24), watch mode.

## Design References

- DESIGN §3.3, §13 (flags/exit codes), §15.1–15.2 (report shapes), §8.2 (missing targets)
