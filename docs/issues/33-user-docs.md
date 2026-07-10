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
  pages listed below, config reference generation script, doc-link wiring from templates
  (Issue 31 placeholders) and PR-body footer (Issue 27).
- Out: contributor docs (Issue 35), API/library docs (CLI-only in v1), translated docs.

## Detailed Requirements

1. `README.md` sections, in order: badges (CI, npm) · what it does (3 bullets + the
   product one-liner EN+JA) · how it works diagram (reuse DESIGN §4 ASCII) · quickstart
   (`npm i -D i18n-agent` → `npx i18n-agent init …` → set key → `npx i18n-agent
   translate`) · GitHub Action quickstart (embed `sync-pr.yml`) · command table
   (from DESIGN §13) · exit-code table · supported formats table · providers table
   (env vars from DESIGN §10.2) · **Security & privacy** (strings sent to configured
   provider; no telemetry; token scopes; fork PR behavior; link to SECURITY.md) ·
   license.
2. `docs/usage/configuration.md`: full reference — one section per config field with
   type/default/constraints, GENERATED from `schemas/config.schema.json` by
   `scripts/gen-config-docs.mjs` (checked in CI for drift like the schema itself);
   hand-written intro + examples for the five formats (Android/iOS path conventions
   from DESIGN §8.6–8.7 exactly).
3. `docs/usage/workflows.md`: W1/W2 setup walkthroughs (embed both templates verbatim
   via includes or copies with a drift-check script), the fork limitation, the
   `pull_request_target` warning (link the anti-pattern doc from Issue 31), cron
   variant snippet, required secrets per provider.
4. `docs/usage/glossary-style.md`: glossary YAML schema with examples (from DESIGN
   §10.5), style-guide authoring guidance, validation severity tuning.
5. `docs/usage/troubleshooting.md`: table of every exit code + the top expected errors
   (missing key env, dirty worktree, UTF-16 strings file, DTD rejection, lockfile
   corruption + `--reset-lockfile`, maxKeysPerRun breaker) each with cause → fix.
6. All CLI invocations in docs are copy-paste runnable; a docs-test script executes the
   quickstart commands against a scratch fixture with the fake provider in CI (drift
   protection for the happy path).
7. Link integrity: `docs`-internal links checked in CI (simple link-checker script; no
   external-link checking).

## Acceptance Criteria

- [ ] README covers all sections incl. Security & privacy; JA summary present.
- [ ] Config reference is generated, drift-checked, and covers 100 % of schema fields.
- [ ] Quickstart executes green in CI via the docs-test script.
- [ ] Templates embedded in docs are drift-checked against `examples/workflows/`.
- [ ] Every DESIGN §13 exit code appears in troubleshooting.

## Validation

- CI: docs-test + drift checks + link checker green.
- Manual: a fresh reader (owner) follows the quickstart on a scratch repo end-to-end.

## Dependencies

28, 29, 30, 31

## Non-goals

Website/docs hosting, versioned docs, localized docs (JA summary only), video/gifs.

## Design References

- DESIGN §13, §14.2, §16 (user-visible posture), §10.2, §8.6–8.7
