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
2. `baseUrl` rule (T-NET): if set, must parse as URL with protocol `https:`, OR host in
   `{localhost, 127.0.0.1, [::1]}`, OR `allowInsecureBaseUrl === true`; otherwise
   `ConfigError`.
3. `load.ts`: `loadConfig(opts: { cwd: string; configPath?: string }): Promise<LoadedConfig>`.
   Steps: resolve path (`configPath` ?? `<cwd>/i18n-agent.config.json`); stat + reject
   files > 256 KiB; read UTF-8; `JSON.parse` (syntax error → `ConfigError` with position);
   zod parse (issues → `ConfigError` listing `path.to.field: message` lines, all issues,
   not just the first).
4. `paths.ts`: `resolveInsideRoot(root: string, p: string): string` — resolves, then
   compares `fs.realpathSync` of the deepest existing ancestor against `realpath(root)`;
   any escape (absolute outside, `..`, symlink out) → `ConfigError` mentioning the
   offending config field. Applied to `path` (template with `{locale}` replaced by a
   probe token first), `sourcePath`, `styleGuidePath`, `glossaryPath`, `lockfilePath`.
5. `LoadedConfig` = `{ config: Config; root: string; configPath: string }`. No API-key
   resolution here (ADR-003 — providers do that).
6. `gen:schema` script: converts the zod schema to JSON Schema (draft 2020-12) via code
   (add the conversion dev-dependency of the implementer's choice from zod's official
   tooling), writes `schemas/config.schema.json` deterministically (sorted keys), and the
   build fails if the file is out of date (`gen:schema --check` in CI).
7. Unit tests (`tests/unit/config/*.test.ts`): happy path minimal + maximal configs;
   every constraint above has a rejecting test; traversal cases `../x`, absolute `/etc`,
   symlink escape (create real temp symlink); baseUrl http-non-localhost rejected,
   localhost allowed, `allowInsecureBaseUrl` override allowed; unknown key rejected;
   duplicate file ids rejected; error message includes field path.

## Acceptance Criteria

- [ ] Schema behavior matches DESIGN §6.2 field-for-field (reviewer diffs table vs code).
- [ ] All five path fields are confined to the repo root incl. symlink escapes.
- [ ] `schemas/config.schema.json` is generated, committed, and drift-checked in CI.
- [ ] `ConfigError` output lists every violation with its config path; exit-code mapping 3
      verified once Issue 03 lands (integration test may live there).
- [ ] 100 % of constraints in this issue have a dedicated unit test.

## Validation

- `npx vitest run tests/unit/config` green.
- `npm run gen:schema && git diff --exit-code schemas/` clean.
- Manual: craft a config with `"lockfilePath": "../../evil.json"` → command aborts with
  `ConfigError` naming `lockfilePath`.

## Dependencies

01

## Non-goals

Config file discovery walking parent dirs (repo root only), YAML/JS config formats,
API-key values in config (forbidden by ADR-003), dynamic plugin fields (ADR-005).

## Design References

- DESIGN §6 (schema/loading), §16 T-PATH, T-NET
- ADR-003, ADR-005
