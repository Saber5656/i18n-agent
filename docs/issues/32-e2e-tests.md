# Title

End-to-end fixture test suite

## Summary

Build the cross-cutting E2E suite: five per-format fixture repos driven through
`check → translate → check` and `pr` (both strategies) using the built CLI binary, the
fake provider, a local bare-repo origin, and nock-mocked GitHub API.

## Context

Unit/integration tests live inside earlier issues; this suite proves the composed product
against the workflows W1–W3 (DESIGN §3, §17) using the real distributed artifact
(`dist/cli.js`), catching wiring/packaging regressions no unit test can.

## Scope

- In: `tests/e2e/**`, fixture repos under `tests/fixtures/e2e/<format>/`, an e2e npm
  script + CI job stage, loop-termination and idempotency scenarios.
- Out: live-API tests (never in CI), real-GitHub Action e2e (DESIGN §19 U5, manual QA in
  Issue 34).

## Detailed Requirements

1. Harness: build once (`npm run build`), then spawn `node dist/cli.js …` via execa in a
   temp copy of each fixture repo (`git init`, commit, add local bare origin). Env:
   `provider.name: "fake"` in fixture configs; no network except nock-intercepted
   GitHub API (nock in the test process CANNOT intercept the child process — so `pr` e2e
   drives the GitHub adapter via an in-process command runner instead: import the CLI
   entry as a module for `pr` cases, spawn-based for pure-file cases; document this
   split in the harness README comment).
2. Scenario matrix (each × 5 formats unless noted):
   - `check` on missing-translations fixture → exit 1, JSON report counts match golden;
   - `translate` → files written match golden bytes; lockfile matches golden; second
     `translate` → `outcome: "clean"`, zero provider work (fake provider call-count file
     trick: fake writes a counter tmp file when `I18N_AGENT_FAKE_COUNTER` env set —
     add this test hook to Issue 14's fake provider spec);
   - stale flow: mutate source value → `check` exit 1 with 1 stale → `translate` →
     fresh;
   - validation rejection: fixture whose fake output breaks placeholders via
     `[[FAKE:FAIL]]`-adjacent scripted stub (json format only) → exit 2, file NOT
     written;
   - `pr --strategy branch` (json + android-xml only): bare origin gains
     `i18n-agent/main`; nock asserts PR create; re-run after no change → no API calls;
   - `pr --strategy commit` (json only): current branch gains one commit; re-run no-op;
   - `validate` on a hand-broken target fixture → exit 2 with expected issue ids;
   - unicode/CJK content survives the full loop byte-exactly (ja targets with emoji).
3. Golden files: stored beside fixtures; an `UPDATE_GOLDEN=1` env regenerates (documented).
4. Runtime budget: whole e2e stage ≤ 3 min in CI (parallel via vitest workers; fixtures
   small).
5. CI: e2e runs as a separate job after build on both Node versions; artifacts
   (tmp-dir trees) uploaded on failure for debugging.

## Acceptance Criteria

- [ ] Full matrix green in CI on node 20 + 22, offline.
- [ ] Loop-termination proven: post-translate runs make zero provider calls (counter
      hook) and zero GitHub API calls (nock strict).
- [ ] Golden-byte comparisons for all five formats incl. lockfile determinism.
- [ ] Suite fails if `dist/cli.js` is missing/stale (guard step) — proves packaging.
- [ ] Failure artifacts uploaded for post-mortem.

## Validation

- `npm run test:e2e` locally green; CI job green twice consecutively (flake check).

## Dependencies

25, 28 (and transitively all format/provider issues); 14 (fake counter hook — coordinate
the small addition there if not yet merged).

## Non-goals

Performance benchmarking, live provider smoke tests, real GitHub integration, Windows
runners (v1 CI is Linux; document).

## Design References

- DESIGN §3 (workflows), §17 (E2E row), §16 T-LOOP verification
