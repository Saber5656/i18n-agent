# Title

Versioning, publishing, provenance, and action tagging

## Summary

Set up changesets-based versioning, the tag-triggered npm publish workflow with
provenance, automated `action.yml` version pinning, major-tag (`v1`) management, and the
pre-release manual QA checklist.

## Context

Release engineering is a security boundary (DESIGN §16 T-SUPPLY, §18): what ships to npm
is what `@v1` action users execute. merge ≠ release is an explicit policy (owner rule +
DESIGN §18).

## Scope

- In: changesets config, `.github/workflows/release.yml`, action version-bump script,
  major-tag move step, `docs/RELEASING.md` incl. manual QA checklist, npm publish
  settings.
- Out: the CI test workflow (Issue 01), community files (Issue 35), first actual release
  (owner-triggered).

## Detailed Requirements

1. Add `@changesets/cli`; config: `access: public`, default bump patch; PRs without a
   changeset fail a CI check unless labeled `no-changeset` (docs-only).
2. `release.yml`: triggered by pushing tag `v*.*.*` (human act). Jobs, in order,
   all actions SHA-pinned, workflow `permissions` minimal per job:
   1. verify: tag matches `package.json` version (abort otherwise); full test suite;
   2. publish: `npm publish --provenance --access public` with
      `permissions: id-token: write` (OIDC) — npm token via repo secret `NPM_TOKEN`
      (registry auth) documented as owner-configured manually (per owner's key-handling
      rule: the agent never provisions secrets);
   3. pin-action: script `scripts/pin-action-version.mjs` rewrites the `EXACT_VERSION`
      literal in `action.yml` to the released version, commits to a `release/pin-<ver>`
      branch, opens a PR (auto-created, human-merged — no direct default-branch push);
   4. move major tag: after the pin PR merges (separate manually-triggered job or
      follow-up step documented in RELEASING.md), force-move `v1` tag to the release
      commit (`git tag -f v1 && git push origin v1 --force` — the ONLY sanctioned force
      push, on a tag in our own repo, documented rationale).
3. `docs/RELEASING.md`: step-by-step owner runbook — changeset flow, version PR, tag,
   the manual QA checklist (from ISSUE_PLAN §6.6): live W1 + W2 run on a scratch repo
   with one real provider + fake provider, results pasted into the release PR;
   npm 2FA/provenance prerequisites listed as owner tasks (DESIGN §19 U8).
4. First-publish note: publish `0.1.0` early after Wave 2 (name-squat mitigation,
   DESIGN §18) — RELEASING.md records this as a recommended owner action.
5. Dry-run job (`workflow_dispatch`): `npm publish --dry-run` + `npm pack` artifact
   upload for inspection without releasing.

## Acceptance Criteria

- [ ] Tag push runs verify → publish → pin PR chain (tested via dry-run mode /
      `workflow_dispatch` path since real publish needs owner secrets).
- [ ] Tag/version mismatch aborts before publish (unit-style test of the verify script).
- [ ] `pin-action-version.mjs` rewrites only the version literal (idempotent, tested).
- [ ] RELEASING.md covers the full runbook incl. QA checklist and owner-only secret
      steps; no step requires the agent to handle secrets.
- [ ] All release workflow actions SHA-pinned; job permissions minimal
      (id-token only where needed).

## Validation

- `workflow_dispatch` dry-run green in CI (uploads pack artifact).
- Script unit tests green; actionlint green.
- Owner review of RELEASING.md (explicit sign-off requested in the issue).

## Dependencies

01, 30

## Non-goals

Actually publishing v1 (owner-triggered), GitHub Releases changelog automation beyond
changesets output, Homebrew/other channels, signing beyond npm provenance.

## Design References

- DESIGN §18, §16 T-SUPPLY, §19 U8
- ADR-001 (version pin contract)
