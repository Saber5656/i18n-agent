# Title

Config schema, loader, and path confinement

## Summary

Implement `i18n-agent.config.json` parsing: the zod schema exactly matching DESIGN §6.2,
the loader with fail-closed validation, repo-root path confinement, JSON Schema
generation, and the `ConfigError` mapping.

## Context

Every command starts here. The config file is semi-trusted input (any contributor can
edit it), so validation and path confinement are security controls, not conveniences
(DESIGN §16 T-PATH, T-NET; ADR-005).

## Scope

- In: `src/config/schema.ts`, `src/config/load.ts`, `src/config/paths.ts`,
  `npm run gen:schema` producing `schemas/config.schema.json`, unit tests.
- Out: consuming the config (commands), `init` scaffolding (Issue 29), glossary file
  parsing (Issue 21).

## Detailed Requirements

1. `schema.ts`: zod schema reproducing DESIGN §6.2 **exactly** — field names, defaults,
   ranges, regexes (`version: z.literal(1)`; locale token `^[A-Za-z]{2,3}(-[A-Za-z0-9]{2,8})*$`;
   file id `^[a-z0-9][a-z0-9-]{0,31}$`; `files` 1..50 with unique ids; `path` contains
   `{locale}` exactly once; `targetLocales` non-empty, excludes `sourceLocale`;
   branchPrefix regex `^[A-Za-z0-9._/-]{1,64}/$`; numeric ranges for concurrency/timeout/
   retries/batchSize/maxKeysPerRun as in §6.2). Use `.strict()` on every object (unknown
   keys rejected). Export inferred type `Config`.
   `files[].options` is a **per-format discriminated union** keyed by `format`:
   `json → { keyStyle?: "nested" | "flat" }` (default `nested`);
   `yaml → { rootLocaleKey?: boolean }` (default `true`);
   `arb | ios-strings | android-xml → {}` (empty object only). Unknown option keys are
   rejected with the offending path.
   `localeMap`: keys must be configured locale tokens; values match
   `^[A-Za-z0-9_-]{1,16}$` (path-segment-safe; no `/`, `.`, or whitespace).
2. `baseUrl` rule (T-NET): if set, must parse as URL with protocol `https:`, OR host in
   `{localhost, 127.0.0.1, [::1]}`, OR `allowInsecureBaseUrl === true`; otherwise
   `ConfigError`.
3. `load.ts`: `loadConfig(opts: { cwd: string; configPath?: string }): Promise<LoadedConfig>`.
   Steps: resolve path (`configPath` ?? `<cwd>/i18n-agent.config.json`); stat + reject
   files > 256 KiB; read UTF-8; `JSON.parse` (syntax error → `ConfigError` with line/
   column); zod parse → `ConfigError` listing **all** issues, one per line, in JSON
   Pointer form: `/files/0/path: must contain "{locale}" exactly once`.
4. `paths.ts`: `resolveInsideRoot(root: string, p: string): string` — resolves, then
   compares `fs.realpathSync` of the deepest existing ancestor against `realpath(root)`;
   any escape (absolute outside, `..`, symlink out) → `ConfigError` mentioning the
   offending config field. Applied to `sourcePath`, `styleGuidePath`, `glossaryPath`,
   `lockfilePath`, and to `path` **expanded per locale**: substitute `{locale}` with
   `sourceLocale` and with every entry of `targetLocales` (after `localeMap` mapping)
   and confine each resulting concrete path — this closes traversal via `localeMap`
   values as well (T-PATH).
5. `LoadedConfig` = `{ config: Config; root: string; configPath: string }`. No API-key
   resolution here (ADR-003 — providers do that).
6. `gen:schema` script: converts the zod schema to JSON Schema (draft 2020-12) via code
   (add the conversion dev-dependency of the implementer's choice from zod's official
   tooling), writes `schemas/config.schema.json` deterministically (sorted keys), and
   supports `--check` (exit 1 on drift, no write). **This issue also edits
   `.github/workflows/ci.yml`** to add a `gen:schema --check` step to the lint job
   (in-scope file change).
7. Unit tests (`tests/unit/config/*.test.ts`): happy path minimal + maximal configs;
   every constraint above has a rejecting test; traversal cases `../x`, absolute `/etc`,
   symlink escape (create real temp symlink); baseUrl http-non-localhost rejected,
   localhost allowed, `allowInsecureBaseUrl` override allowed; unknown key rejected;
   duplicate file ids rejected; error message includes field path.

## Acceptance Criteria

- [ ] `tests/unit/config/schema.test.ts` contains a constraint matrix: one accepting and
      one rejecting case per DESIGN §6.2 constraint named in requirement 1 (table-driven;
      the test names enumerate the constraints).
- [ ] All five path fields are confined to the repo root incl. symlink escapes and
      per-locale template expansion (incl. a malicious `localeMap` value case).
- [ ] `schemas/config.schema.json` is generated, committed, snapshot-tested, and
      drift-checked in CI via `gen:schema --check`.
- [ ] `ConfigError` output lists every violation in JSON Pointer form (multi-error
      fixture asserted); `ConfigError.code === 3` asserted (class from Issue 03).

## Validation

- `npx vitest run tests/unit/config` green.
- `npm run gen:schema && git diff --exit-code schemas/` clean.
- Manual: craft a config with `"lockfilePath": "../../evil.json"` → command aborts with
  `ConfigError` naming `lockfilePath`.

## Dependencies

01, 03 (`ConfigError` base class and exit-code table)

## Non-goals

Config file discovery walking parent dirs (repo root only), YAML/JS config formats,
API-key values in config (forbidden by ADR-003), dynamic plugin fields (ADR-005).

## Design References

- DESIGN §6 (schema/loading), §16 T-PATH, T-NET
- ADR-003, ADR-005
