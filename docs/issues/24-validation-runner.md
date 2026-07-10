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
     severities: Config["validation"]; arbPlaceholderNames?: string[] }): ValidationVerdict
   ```
   Mapping: each validator's raw issues get severity from config
   (`placeholders|tags|icuSyntax|empty|glossary`); `"off"` skips the validator entirely;
   any final `error` ⇒ `accepted: false` (DESIGN §11.1 rejection semantics — entry not
   written, lockfile untouched, reported failed).
2. `validate` command flow: config → resolve paths → parse source+targets → for every
   target entry that also exists in source, run `validateEntry`
   (glossary loaded via Issue 21; profiles = file override ?? adapter defaults) →
   `RunReport` (`command: "validate"`, `outcome: "clean" | "partial"`), console/JSON via
   Issue 13 renderers; issues listed per file×locale. Exit 2 if any error-severity issue,
   else 0 (warn-only → 0). `--locale/--file` filters as in Issue 13.
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
- [ ] `validate` is read-only (tree-hash test) and provider-free (no network in test env).
- [ ] Reports render through the shared `RunReport` machinery (no parallel format).

## Validation

- `npx vitest run tests/unit/validate/runner.test.ts tests/integration/validate.test.ts`
  green.

## Dependencies

02, 13, 22, 23

## Non-goals

Auto-fixing issues, validating source-locale quality, per-entry severity overrides
(config is global per validator in v1).

## Design References

- DESIGN §11.1 (runner semantics), §3.3 (W3), §13 (exit codes), §15.2 (report)
