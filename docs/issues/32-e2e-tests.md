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
   `provider.name: "fake"` in fixture configs. Network discipline: every test file
   calls `nock.disableNetConnect()` (allowing only `127.0.0.1` for nock itself) and
   fails on any unmatched request (strict mode); spawned-CLI cases run with fixture
   configs that need no network at all (fake provider), while `pr` cases drive the
   GitHub adapter via an in-process command runner (import the CLI entry as a module)
   so nock can intercept — this split is documented in the harness README comment.
   Secret-leak assertions: harness env sets `FAKE_API_KEY=sk-e2e-sentinel-123456` and
   `GITHUB_TOKEN=ghs_e2e-sentinel-654321`; after every scenario, captured stdout,
   stderr, JSON reports, and nock-recorded PR bodies are grepped to assert neither
   sentinel appears (T-SECRET end-to-end).
2. Scenario matrix (each × 5 formats unless noted):
   - `check` on missing-translations fixture → exit 1, JSON report counts match golden;
   - `translate` → files written match golden bytes; lockfile matches golden; second
     `translate` → `outcome: "clean"`, zero provider calls asserted via the fake
     provider's `I18N_AGENT_FAKE_COUNTER` hook (specified in Issue 14);
   - stale flow: mutate source value → `check` exit 1 with 1 stale → `translate` →
     fresh;
   - validation rejection: fixture whose fake output breaks placeholders via
     `[[FAKE:FAIL]]`-adjacent scripted stub (json format only) → exit 2, file NOT
     written;
   - `pr --strategy branch` (json + android-xml only): bare origin gains
     `i18n-agent/main`; nock asserts PR create; commit message trailer
     `X-i18n-agent:` asserted via `git log` in the bare repo; re-run after no change →
     no API calls; source-change re-run → lease force-push + PR **update** (nock PATCH,
     not POST); a direct force-push attempt to a non-prefix branch through the git
     layer is rejected (guard test at e2e level);
   - `pr --strategy commit` (json only): current branch gains one commit with the
     trailer; re-run no-op;
   - `validate` on a hand-broken target fixture → exit 2 with expected issue ids;
   - unicode/CJK content survives the full loop byte-exactly (ja targets with emoji).
3. Fixture/golden layout (normative): `tests/fixtures/e2e/<format>/repo/**` (the input
   repo incl. config), `tests/fixtures/e2e/<format>/golden/after-translate/**`
   (expected file bytes incl. lockfile), `tests/fixtures/e2e/<format>/golden/reports/
   <scenario>.json` (expected `--json` reports, volatile fields normalized). Byte
   comparison covers locale files + lockfile; `.git` and timestamps excluded.
   `UPDATE_GOLDEN=1` regenerates (documented).
4. Runtime budget: whole e2e stage ≤ 3 min in CI (parallel via vitest workers; fixtures
   small).
5. CI: e2e job runs `npm run test:e2e && npm run test:e2e` (two consecutive full runs —
   the objective flake/determinism gate) after build on both Node versions; artifacts
   (tmp-dir trees) uploaded on failure for debugging.

## Acceptance Criteria

- [ ] Full matrix green in CI on node 20 + 22, offline (`nock.disableNetConnect` +
      strict unmatched-request failure in every e2e file).
- [ ] Loop-termination proven: post-translate runs make zero provider calls (counter
      hook) and zero GitHub API calls (nock strict).
- [ ] PR branch scenarios cover create AND update paths, the trailer, and the
      force-guard rejection.
- [ ] Secret sentinels never appear in any captured output or PR body.
- [ ] Golden-byte comparisons for all five formats incl. lockfile determinism, using
      the normative fixture layout.
- [ ] Suite fails if `dist/cli.js` is missing/stale (guard step) — proves packaging.
- [ ] Failure artifacts uploaded for post-mortem; CI runs the suite twice back-to-back.

## Validation

- `npm run test:e2e` locally green; CI e2e job (double run) green.

## Dependencies

25, 28 (and transitively all format/provider issues; the fake provider's
`I18N_AGENT_FAKE_COUNTER` hook is part of Issue 14's spec).

## Non-goals

Performance benchmarking, live provider smoke tests, real GitHub integration, Windows
runners (v1 CI is Linux; document).

## Design References

- DESIGN §3 (workflows), §17 (E2E row), §16 T-LOOP verification
