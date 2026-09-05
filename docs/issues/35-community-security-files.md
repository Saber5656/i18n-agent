# Title

Community and security policy files, repo hygiene

## Summary

Add the OSS health files — SECURITY.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, issue/PR
templates — plus Dependabot configuration and repository settings documentation.

## Context

Required for a public OSS release posture (task requirement; DESIGN §16 T-SUPPLY,
SECURITY.md mandate). Kept separate from code so it can land any time after scaffold.

## Scope

- In: `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`,
  `.github/ISSUE_TEMPLATE/{bug_report.yml,feature_request.yml,config.yml}`,
  `.github/PULL_REQUEST_TEMPLATE.md`, `.github/dependabot.yml`,
  `docs/REPO_SETTINGS.md`.
- Out: release runbook (Issue 34), user docs (Issue 33), enabling GitHub org settings
  (owner-manual, documented only).

## Detailed Requirements

1. `SECURITY.md`: supported-versions table (v1 line only), private reporting channel =
   GitHub Security Advisories ("Report a vulnerability" — no email required),
   response-time expectation (best-effort 14 days), scope notes (what counts: T-* rows
   of DESIGN §16; out of scope: translation quality), safe-harbor sentence for
   good-faith research.
2. `CONTRIBUTING.md`: dev setup (`npm ci`, node ≥20), command cheatsheet (lint/typecheck/
   test/build/e2e), PR rules (changeset required, CI green, no force-push to shared
   branches), adapter/provider contribution pointers to DESIGN §8/§10 + conformance
   harness, DCO-style sign-off NOT required (keep friction low; MIT + inbound=outbound
   noted).
3. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 verbatim; enforcement contact
   split: conduct reports → GitHub Issues (public) or the maintainer contact
   placeholder `@Saber5656`; security vulnerabilities → GitHub Security Advisories
   only (never public issues).
4. Issue forms (YAML): bug report (version, command, config with secrets-redaction
   warning, expected/actual, logs) and feature request (problem, proposal,
   alternatives); `config.yml` exact contents: `blank_issues_enabled: false` and one
   `contact_links` entry pointing to SECURITY.md for vulnerability reports.
5. PR template: summary, linked issue (`Closes #NN`), checklist (tests added, changeset
   added or `no-changeset` justified, docs touched if user-facing).
6. `dependabot.yml`: weekly **version updates** for `npm` + `github-actions`
   ecosystems, grouped minor/patch. (Dependabot *security* updates are a repository
   setting, not a schedule in this file — documented in REPO_SETTINGS.md instead.)
7. `docs/REPO_SETTINGS.md`: documented owner-applied settings — default-branch ruleset
   (PR required, no force push, no direct push), secret scanning + push protection ON,
   **Dependabot alerts ON and Dependabot security updates ON** (Settings → Code
   security and analysis), Actions permissions read-only default, tag protection for
   `v*`. Each with the GitHub UI path. (Agent documents; owner applies — key/settings
   handling stays manual per owner policy.)

## Acceptance Criteria

- [ ] All files present in expected locations (`git ls-files` list in PR); internal
      links resolve via the Issue 33 link checker (or a local equivalent if 33 is not
      yet merged).
- [ ] YAML validity of issue forms + dependabot.yml proven by a small parse script
      (`scripts/validate-github-yaml.mjs` using the already-allowlisted `yaml`
      package) run in CI; rendered-form screenshots from a pushed branch attached to
      the PR as evidence.
- [ ] SECURITY.md names the Advisories flow and scopes it to DESIGN §16 threat classes.
- [ ] Dependabot config: weekly version updates, both ecosystems, grouped minor/patch;
      no security-update schedule inside the file.
- [ ] REPO_SETTINGS.md lists every setting incl. Dependabot alerts/security updates
      with UI paths; no step requires agent access to secrets.
- [ ] CoC contact split (conduct vs vulnerability) present.

## Validation

- `node scripts/validate-github-yaml.mjs` green in CI; screenshots of both issue forms
  and the PR template rendering attached to the implementation PR.

## Dependencies

01

## Non-goals

Funding files, all-contributors bot, Discussions setup, trademark/branding work,
applying the GitHub settings themselves (owner-manual).

## Design References

- DESIGN §16 (SECURITY.md mandate, T-SUPPLY), §18
