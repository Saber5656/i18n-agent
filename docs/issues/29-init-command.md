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
   `--path <template>` (default per format, e.g. json → `locales/{locale}.json`,
   android-xml → `app/src/main/res/values-{locale}/strings.xml` plus `sourcePath`
   comment), `--source-locale <l>` (default `en`), `--target-locales <l,l,…>`
   (required — no default; `UsageError` listing an example when absent),
   `--provider <name>` (default `openai`), `--force`.
2. Output file: `<cwd>/i18n-agent.config.json`. Exists and no `--force` → `UsageError`
   (exit 3) "already exists; use --force". With `--force`: overwrite.
3. Generated config MUST validate against Issue 02's schema (test round-trips it through
   `loadConfig`) and includes: `$schema` URL (DESIGN §6.2), `version: 1`, the flag-driven
   fields, a `provider.model` placeholder value `"CHANGE-ME"` (schema-valid non-empty,
   obviously actionable), default validation/git blocks omitted (rely on schema
   defaults — keep the file minimal).
4. After writing, print next steps (stderr, human-oriented): set provider API key env
   var (name resolved per provider), replace `provider.model`, run `i18n-agent check`.
   `--json` prints `{ created: true, path: … }` to stdout instead.
5. android-xml template additionally sets `sourcePath` and a `localeMap` example (values
   from DESIGN §8.7) as real fields.
6. Never writes anything else (no directories, no sample locale files).
7. Tests: default json init validates via loadConfig; every format template validates;
   exists/no-force error; force overwrite; missing target-locales error text; JSON mode
   output shape; generated file byte-stable (snapshot).

## Acceptance Criteria

- [ ] Every generated template passes `loadConfig` untouched except `model` placeholder
      (which is schema-valid).
- [ ] Overwrite protection + `--force` behavior tested.
- [ ] Output file is minimal (schema defaults not duplicated) and deterministic.
- [ ] Next-steps text names the correct env var per chosen provider.

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
