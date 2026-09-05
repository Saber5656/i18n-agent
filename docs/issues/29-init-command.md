# Title

`init` command

## Summary

Implement `i18n-agent init`: non-interactive scaffolding of a valid
`i18n-agent.config.json` from flags, with safe-overwrite behavior and next-steps output.

## Context

First-touch UX (DESIGN §3.3, §13). Non-interactive by design so it behaves identically
in CI and local shells (interactive wizard is v2, DESIGN §2.3).

## Scope

- In: `src/cli/commands/init.ts`, config template generation, unit/integration tests.
- Out: format auto-detection of existing repos (v2), config schema changes.

## Detailed Requirements

1. Flags: `--format <json|yaml|arb|ios-strings|android-xml>` (default `json`),
   `--path <template>` (default per format, e.g. json → `locales/{locale}.json`;
   android-xml → `app/src/main/res/values-{locale}/strings.xml` with real JSON field
   `sourcePath: "app/src/main/res/values/strings.xml"` — JSON has no comments, so
   everything is expressed as actual fields), `--source-locale <l>` (default `en`),
   `--target-locales <l,l,…>` (required — no default; `UsageError` listing an example
   when absent), `--provider <openai|anthropic|gemini|ollama>` (default `openai`;
   flag is in DESIGN §13), `--force`.
2. Output file: `<cwd>/i18n-agent.config.json`. Exists and no `--force` → `UsageError`
   (exit 3) "already exists; use --force". With `--force`: overwrite.
3. Generated config MUST pass `loadConfig` **before being written** (validation failure
   — e.g. a traversal/absolute `--path` template — writes nothing and exits 3; this is
   the T-PATH gate for user-supplied templates). Exact generated top-level keys, and no
   others: `$schema` (DESIGN §6.2 URL), `version: 1`, `sourceLocale`, `targetLocales`,
   `files` (one entry: `id: "main"`, `format`, `path`, plus `sourcePath` and
   `localeMap` only for android-xml), `provider` (`name` + `model: "CHANGE-ME"` —
   schema-valid, obviously actionable). All other blocks omitted (schema defaults).
   Committed snapshot fixtures per format are the normative expected outputs.
4. After writing, print next steps (stderr, human-oriented): set provider API key env
   var (name resolved per provider; ollama → "no key needed"), replace
   `provider.model`, run `i18n-agent check`. `--json` mode: stdout gets exactly
   `{ "created": true, "path": "<cwd-relative path>" }` and the stderr next-steps are
   suppressed.
5. android-xml `localeMap` generation rule: include the field only when at least one
   `--target-locales` entry contains a region subtag (`xx-YY`); map each such locale to
   the Android form (`zh-CN → zh-rCN`, `pt-BR → pt-rBR` — insert `r` before the
   region); never include locales that were not requested.
6. Never writes anything else (no directories, no sample locale files).
7. Tests: default json init validates via loadConfig; every format template matches its
   committed snapshot AND validates; exists/no-force error; force overwrite; missing
   target-locales error text; traversal `--path` → exit 3, nothing written; localeMap
   rule matrix (`ja` alone → no map; `zh-CN` → mapped); JSON mode stdout schema +
   stderr silence; per-provider next-steps env var names.

## Acceptance Criteria

- [ ] Every generated template passes `loadConfig` before write; invalid templates
      write nothing (T-PATH test).
- [ ] Overwrite protection + `--force` behavior tested.
- [ ] Generated files match committed snapshots (normative minimal key set).
- [ ] `--json` stdout schema exact; stderr suppressed in JSON mode.
- [ ] Next-steps text names the correct env var per chosen provider (incl. ollama's
      no-key message).
- [ ] localeMap generation rule matrix tested.

## Validation

- `npx vitest run tests/integration/init.test.ts` green.
- Manual: `i18n-agent init --format yaml --target-locales ja` then `i18n-agent check`
  runs without config errors on a fixture repo.

## Dependencies

02, 03

## Non-goals

Interactive prompts, repo scanning/auto-detection, glossary/style file scaffolding,
workflow file generation (Issue 31 documents that path).

## Design References

- DESIGN §3.3, §6.2, §13, §2.3 (wizard deferral)
