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

1. Add `@changesets/cli`; config: `access: public`, default bump patch. **This issue
   edits `.github/workflows/ci.yml`** to add a `changeset-check` job: PRs without a
   changeset fail unless the PR carries the `no-changeset` label (docs-only changes).
2. `release.yml`: triggered by pushing tag `v*.*.*` (human act). Jobs, in order, all
   actions SHA-pinned, with **explicit per-job permissions**:
   1. verify (`permissions: contents: read`): tag ref parses as semver AND equals
      `package.json` version (abort otherwise); full test suite;
   2. publish (`permissions: contents: read, id-token: write` — OIDC provenance):
      `npm publish --provenance --access public`; npm registry auth via repo secret
      `NPM_TOKEN`, documented as owner-configured manually (per owner's key-handling
      rule: the agent never provisions secrets);
   3. pin-action (`permissions: contents: write, pull-requests: write`): script
      `scripts/pin-action-version.mjs` rewrites the version literal in `action.yml` —
      the ONLY accepted pattern is the string `i18n-agent@<semver-or-placeholder>`
      inside the run step; the script fails if it finds zero or more than one match —
      commits to a `release/pin-<ver>` branch and opens a PR (auto-created,
      human-merged — no direct default-branch push);
   4. move major tag (manually-triggered `workflow_dispatch` job, documented in
      RELEASING.md; `permissions: contents: write`): **after the pin PR merges**,
      force-move `v1` to the **pin-PR merge commit on the default branch** — NOT the
      original tag commit — so `@v1` users always get the action with the reviewed
      `EXACT_VERSION` baked in (DESIGN §14.1). This tag move is the ONLY sanctioned
      force push (a tag in our own repo; rationale documented inline).
3. `docs/RELEASING.md`: step-by-step owner runbook — changeset flow, version PR, tag,
   the manual QA checklist (ISSUE_PLAN §6 item 6 / DESIGN §17): live W1 + W2 run on a
   scratch repo with one real provider + fake provider, plus a `package-spec` canary
   run of the action, results pasted into the release PR; npm 2FA/provenance
   prerequisites listed as owner tasks (DESIGN §19 U8).
4. First-publish note: publish `0.1.0` early after Wave 2 (name-squat mitigation,
   DESIGN §18) — RELEASING.md records this as a recommended owner action.
5. Dry-run job (`workflow_dispatch`): `npm publish --dry-run` + `npm pack` artifact
   upload for inspection without releasing.

## Acceptance Criteria

- [ ] Three separately-tested release-path pieces (real publish needs owner secrets, so
      each is unit/dry-tested): (a) tag-parse/version-match verify script — unit tests
      incl. mismatch abort and non-semver tag; (b) publish command construction —
      asserted via `npm publish --dry-run` in the `workflow_dispatch` dry-run job;
      (c) pin-PR path — script unit tests + a dry mode that prints the would-be branch
      name and diff without pushing.
- [ ] `pin-action-version.mjs`: rewrites exactly one `i18n-agent@…` occurrence, is
      idempotent, and fails on zero/multiple matches (all unit-tested).
- [ ] `v1` tag-move job targets the pin-PR merge commit (job takes the merge SHA as a
      required input; documented in RELEASING.md).
- [ ] `changeset-check` CI job blocks changeset-less PRs without the `no-changeset`
      label (verified on a test PR).
- [ ] RELEASING.md covers the full runbook incl. QA checklist and owner-only secret
      steps; no step requires the agent to handle secrets.
- [ ] All release workflow actions SHA-pinned; the exact per-job permissions blocks
      from requirement 2 are present (policy-test grep).

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
