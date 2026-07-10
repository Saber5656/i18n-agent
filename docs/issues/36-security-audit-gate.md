# Title

Security acceptance audit against the threat model

## Summary

Run a systematic pre-release audit: verify every DESIGN §16 threat row has a passing
automated test or a documented reviewed control, add any missing security tests, and
produce the audit record.

## Context

Individual issues each carry their own security acceptance criteria; this gate issue
prevents "death by a thousand skipped checkboxes" by re-verifying the whole threat model
against the actual codebase before v1 (ISSUE_PLAN §6.5). It is intentionally the last
issue.

## Scope

- In: audit of all 14 threat rows (T-INJ … T-PRIV), a consolidated
  `tests/security/` suite collecting/adding the adversarial fixtures, the audit report
  `docs/security-audit-v1.md`, fixes for gaps found (small fixes in-issue; large gaps →
  new issues per ISSUE_PLAN §8 discovery rule).
- Out: new features, threat-model expansion (changes to §16 go through DESIGN + ADR
  updates first).

## Detailed Requirements

1. For each DESIGN §16 row, record in `docs/security-audit-v1.md`:
   threat id · control summary · verification type (`automated-test` | `reviewed-control`)
   · exact test path(s) or code location · verdict (pass/gap) · notes.
2. `tests/security/` consolidation — these minimum adversarial cases must exist and pass
   (create any missing ones here):
   - T-INJ: locale file whose values contain instruction-injection strings → prompt
     user-payload remains data (re-parse assertion) AND end-to-end fake-provider run
     never emits them outside translation values;
   - T-SECRET: forced provider error embedding the key; PR body render; `--json` report;
     verbose logs — all show `***`/no key (grep the captured outputs);
   - T-XXE: DOCTYPE/XXE/billion-laughs fixtures rejected, no fs read of payload path;
   - T-YAML: alias bomb < 1 s rejection;
   - T-PATH: traversal + symlink-escape configs rejected for all five path fields;
   - T-CMD: ref/path injection attempts rejected (from Issue 26 suite, referenced);
   - T-FORK/T-PRIV: actionlint + a policy test asserting templates contain the same-repo
     guard string, minimal permissions, and no `pull_request_target` anywhere in the
     repo (`git grep` test);
   - T-LOOP: e2e second-run zero-provider-call + recursion-guard tests (referenced);
   - T-FORCE: prefix guard test (referenced);
   - T-REDOS: timing-budget tests (referenced);
   - T-SUPPLY: CI job asserting all workflow `uses:` are 40-char SHA pinned; dependency
     allowlist test — `package.json` runtime deps ⊆ DESIGN §5 list;
   - T-NET: non-https non-localhost baseUrl rejection (referenced);
   - T-LOCK: corrupt-lockfile fixtures (referenced).
3. Referenced tests are asserted present by path in the audit doc (no duplication);
   missing ones are written here.
4. Run `npm audit --omit=dev` and record the result; high/critical findings in runtime
   deps → fix or file dedicated issues before closing.
5. Gap protocol: any control that fails verification → fix here if ≤ ~50 LOC test/code
   change, otherwise open a new `docs/issues/NN-*.md` + GitHub Issue and link it in the
   audit doc. The audit doc's final section lists residual accepted risks (must match
   DESIGN §16's residual-risk statements; discrepancies require a DESIGN update PR).

## Acceptance Criteria

- [ ] `docs/security-audit-v1.md` covers all 14 threat ids with verdicts and pointers;
      zero unresolved gaps (fixed or spun into linked issues).
- [ ] `tests/security/` suite green in CI and wired into the standard test run.
- [ ] Repo-wide policy greps (no `pull_request_target`, SHA-pinned actions, dep
      allowlist) run as CI tests, not one-off scripts.
- [ ] `npm audit` result recorded; no unaddressed high/critical runtime findings.
- [ ] Residual-risk list matches DESIGN §16 (or DESIGN updated via PR).

## Validation

- `npx vitest run tests/security` green in CI.
- Owner reviews and signs off the audit doc in the closing PR (explicit request in PR
  description).

## Dependencies

02, 03, 07, 10, 15, 25, 26, 31 (audits their controls; runs last — after 32 in practice)

## Non-goals

External penetration testing, formal verification, expanding the threat model, auditing
GitHub org settings (documented owner tasks in Issue 35).

## Design References

- DESIGN §16 (entire), §17 (security fixtures row)
- ISSUE_PLAN §6 (validation strategy), §8 (discovery rule)
