# Title

Validation runner and `validate` command

## Summary

Implement the severity-applying validation runner used by the translate pipeline, and the
standalone `i18n-agent validate` command that audits existing catalogs.

## Context

The runner is the policy point where config severities decide accept/reject/warn for
each translated entry (DESIGN §11.1); `validate` exposes the same machinery as a CI
quality gate over human-edited translations (workflow W3).

## Scope

- In: `src/validate/runner.ts`, `src/cli/commands/validate.ts`, integration tests.
- Out: individual validators (22/23), translate pipeline wiring (Issue 25 calls the
  runner).

## Detailed Requirements

1. Runner API:
   ```ts
   export interface ValidationVerdict { accepted: boolean;
     issues: ValidationIssue[] }   // issues carry final severity after config mapping
   validateEntry(input: { source: string; translated: string;
     profiles: PlaceholderProfileId[]; glossary: GlossaryTerm[];
     fileId: string; key: string; locale: string;
     severities: Config["validation"];
     placeholderHints?: string[]   // adapter-neutral; runner fills it from
                                   // entry.meta.placeholderNames when present (ARB)
   }): ValidationVerdict
   ```
   Mapping: each validator's raw issues get severity from config
   (`placeholders|tags|icuSyntax|empty|glossary`); `"off"` skips the validator entirely;
   any final `error` ⇒ `accepted: false` (DESIGN §11.1 rejection semantics — entry not
   written, lockfile untouched, reported failed).
2. `validate` command flow: config → resolve paths → parse source+targets → for every
   target entry that also exists in source, run `validateEntry`
   (glossary loaded via Issue 21; profiles = file override ?? adapter defaults) →
   `RunReport` (`command: "validate"`), console/JSON via Issue 13 renderers.
   Report mapping (normative, one committed JSON example per case in the test
   fixtures): clean → `outcome: "clean"`, all counts 0, `issues: []`, exit 0;
   warn-only → `outcome: "clean"`, warn issues in `files[].issues`, `counts.failed: 0`,
   exit 0; errors → `outcome: "partial"`, each error-bearing entry counted in
   `counts.failed` and listed in `failures` (`reason: "validation"`), exit 2.
   Filtered-out files/locales do not appear in `files[]`.
   `--locale/--file` filter rules (restated from Issue 13, same behavior): unknown
   locale/file id → `UsageError` exit 3 listing configured values; repeated flags
   deduped; filters matching zero pending pairs → clean report, exit 0.
3. `validate` never contacts providers, never writes files (read-only guarantee like
   `check`).
4. Orphan/missing entries are NOT validation targets (diff's domain — avoid duplicate
   reporting; document).
5. Integration tests: fixture with seeded violations per validator class × severity
   configs (error/warn/off) asserting exit codes and report contents; read-only
   assertion; ARB placeholder-hint path exercised end-to-end (arb fixture with
   `placeholders` metadata and a target missing the token).

## Acceptance Criteria

- [ ] Severity mapping table behavior (error/warn/off × 5 validators) fully tested via
      the runner unit tests.
- [ ] Rejection semantics verified: an `error` verdict marks `accepted: false` and the
      command reports it as failure with exit 2.
- [ ] The three committed report examples (clean / warn-only / errors) match actual
      `--json` output byte-for-byte (`satisfies RunReport` + snapshot).
- [ ] `validate` is read-only (tree-hash test) and provider-free: the test asserts the
      provider registry module is never imported/instantiated (module spy) and runs
      with network disabled (`nock.disableNetConnect()` or equivalent).
- [ ] Console output rendered via `report/console.ts` (integration test asserts the
      shared renderer is invoked, not a parallel formatter).

## Validation

- `npx vitest run tests/unit/validate/runner.test.ts tests/integration/validate.test.ts`
  green.

## Dependencies

02, 13, 21 (glossary loader used by the command), 22, 23

## Non-goals

Auto-fixing issues, validating source-locale quality, per-entry severity overrides
(config is global per validator in v1).

## Design References

- DESIGN §11.1 (runner semantics), §3.3 (W3), §13 (exit codes), §15.2 (report)
