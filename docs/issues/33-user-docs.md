# Title

README and user documentation

## Summary

Write the English README and `docs/` user documentation set: quickstart, configuration
reference generated from the schema, workflow setup guides for W1/W2, provider setup,
security/privacy notes, and troubleshooting.

## Context

Docs are the product surface for an OSS tool; the security posture (fork limits, token
scopes, privacy of strings sent to providers) must be user-visible, not just designed
(DESIGN §14.2, §16). The current README is a one-line Japanese placeholder.

## Scope

- In: `README.md` (EN, with a short 日本語 summary section at the end), `docs/usage/`
  pages listed below, config reference generation script, and doc-link wiring in
  exactly three places: the `DOCS_URL` constant in `src/report/prBody.ts` (Issue 27
  left a placeholder), and the docs-link comment placeholder in each of the two
  Issue 31 templates — mechanical one-line substitutions, listed file-by-file.
- Out: contributor docs (Issue 35), API/library docs (CLI-only in v1), translated docs,
  any other code change.

## Detailed Requirements

1. `README.md` sections, in order: badges (CI, npm) · what it does (3 bullets + the
   product one-liner EN+JA) · how it works diagram (reuse DESIGN §4 ASCII) · quickstart
   (`npm i -D i18n-agent` → `npx i18n-agent init …` → set key → `npx i18n-agent
   translate`) · GitHub Action quickstart (embed `sync-pr.yml`) · command table
   (from DESIGN §13) · exit-code table · supported formats table · providers table
   (env vars from DESIGN §10.2) · **Security & privacy** (strings sent to configured
   provider; no telemetry; token scopes; fork PR behavior; link to SECURITY.md) ·
   license.
2. `docs/usage/configuration.md`: hand-written intro + five per-format examples
   (Android/iOS path conventions from DESIGN §8.6–8.7 exactly), then a generated
   reference section delimited by `<!-- gen:config-reference:start -->` /
   `<!-- gen:config-reference:end -->` markers, produced by
   `scripts/gen-config-docs.mjs` from `schemas/config.schema.json`: one table row per
   schema property (recursed depth-first, JSON-Pointer sort order) with columns
   `Field | Type | Default | Constraints | Description`; enum values and nullability
   rendered into Type; `--check` mode exits 1 when the committed region differs
   (drift gate). Coverage rule: every schema property appears exactly once (the
   generator fails if a property would be skipped).
3. `docs/usage/workflows.md`: W1/W2 setup walkthroughs embedding both templates as
   **copies** inside `<!-- gen:workflow:sync-pr:start/end -->`-style marker pairs;
   `scripts/check-docs-sync.mjs --check` fails when an embedded copy differs from its
   source under `examples/workflows/` (regenerates without `--check`). Plus: the fork
   limitation, the `pull_request_target` warning (link the anti-pattern doc from Issue
   31), cron variant snippet, required secrets per provider.
4. `docs/usage/glossary-style.md`: glossary YAML schema with examples (from DESIGN
   §10.5), style-guide authoring guidance, validation severity tuning.
5. `docs/usage/troubleshooting.md`: table of every exit code + the top expected errors
   (missing key env, dirty managed paths in commit strategy, UTF-16 strings file, DTD
   rejection, lockfile corruption + `--reset-lockfile`, maxKeysPerRun breaker) each
   with cause → fix. Provider setup docs must include the T-NET rules: `baseUrl`
   https-required, localhost exception, `allowInsecureBaseUrl`, and the risk of routing
   strings to a rogue endpoint (DESIGN §16 T-NET).
6. Concrete scripts + CI wiring (all added to package.json and the ci.yml `docs` job by
   this issue): `npm run docs:test` (executes the README quickstart commands against a
   scratch fixture with the fake provider), `npm run docs:check-config`
   (`gen-config-docs.mjs --check`), `npm run docs:check-workflows`
   (`check-docs-sync.mjs --check`), `npm run docs:check-links` (internal-link checker
   script; external links excluded).
7. The owner quickstart walkthrough is release-QA evidence (Issue 34's checklist), not
   a completion gate for this issue.

## Acceptance Criteria

- [ ] README covers all sections incl. Security & privacy (with T-NET baseUrl rules);
      JA summary present.
- [ ] Config reference generated between markers, drift-checked, generator fails on
      skipped properties (coverage rule test).
- [ ] Quickstart executes green in CI via `npm run docs:test`.
- [ ] Embedded templates byte-match `examples/workflows/` via
      `npm run docs:check-workflows`.
- [ ] Every DESIGN §13 exit code appears in troubleshooting.
- [ ] The three doc-link substitutions (prBody `DOCS_URL`, two template placeholders)
      are done and nothing else in `src/` changed (diff-scope check in PR review).

## Validation

- CI `docs` job: `npm run docs:test && npm run docs:check-config &&
  npm run docs:check-workflows && npm run docs:check-links` green.

## Dependencies

28, 29, 30, 31

## Non-goals

Website/docs hosting, versioned docs, localized docs (JA summary only), video/gifs.

## Design References

- DESIGN §13, §14.2, §16 (user-visible posture), §10.2, §8.6–8.7
