# Title

Repository scaffold, toolchain, and CI

## Summary

Create the buildable, testable, lintable skeleton of the `i18n-agent` npm package:
package metadata, TypeScript/ESM setup, tsup build, vitest, biome, directory skeleton,
license, and the repository's own CI workflow.

## Context

Every other issue assumes this toolchain exists (DESIGN §5, §17). Nothing product-specific
is implemented here beyond an executable placeholder CLI entry.

## Scope

- In: package.json, tsconfig, tsup, vitest, biome, `.gitignore`, `.editorconfig`,
  LICENSE (MIT), `src/`+`tests/` skeleton dirs, placeholder `src/cli/index.ts`,
  `.github/workflows/ci.yml`, Node engines.
- Out: any command logic, config parsing, release workflow (Issue 34), community files
  (Issue 35).

## Detailed Requirements

1. `package.json`: `"name": "i18n-agent"`, `"version": "0.0.0"`, `"type": "module"`,
   `"license": "MIT"`, `"engines": { "node": ">=20" }`, `"bin": { "i18n-agent": "dist/cli.js" }`,
   `"files": ["dist", "schemas", "README.md", "LICENSE"]` (Issue 30 appends `action.yml`
   when it exists),
   scripts: `build` (tsup), `test` (vitest run), `test:watch`, `lint` (biome check .),
   `format` (biome format --write .), `typecheck` (tsc --noEmit), `gen:schema`
   (placeholder that exits 0 until Issue 02 implements it).
2. Runtime dependencies: **install none yet.** Dev dependencies: `typescript`, `tsup`,
   `vitest`, `@biomejs/biome`. Runtime deps are added by the issue that first uses them,
   drawn only from the allowlist in DESIGN §5.
3. `tsconfig.json`: `strict: true`, `module`/`moduleResolution` `NodeNext`, `target ES2022`,
   `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.
4. `tsup.config.ts`: entry `src/cli/index.ts`, format esm, node20 target, shebang banner
   `#!/usr/bin/env node`, `dist/` output, sourcemaps.
5. `src/cli/index.ts` placeholder: prints `i18n-agent <version> — not implemented yet` and
   exits 0. `node dist/cli.js` must run after build.
6. Directory skeleton per DESIGN §5 (empty `.gitkeep` files where needed):
   `src/{cli/commands,config,core,formats,providers,validate,glossary,git,report,util}`,
   `tests/{unit,integration,e2e,fixtures}`, `examples/workflows`, `schemas/`.
7. `.github/workflows/ci.yml`: triggers `push: branches: [main]` + `pull_request`;
   jobs: lint → typecheck → test → build, plus an `actionlint` job linting
   `.github/workflows/*.yml` (DESIGN §17 static row; pinned actionlint action or
   pinned binary download); matrix node `20`, `22`;
   **all `uses:` actions pinned by full commit SHA** with a version comment (DESIGN §16
   T-SUPPLY); `permissions: contents: read` at workflow level.
8. `.gitignore`: node_modules, dist, coverage, `.env*`, OS junk. Commit `package-lock.json`.
9. LICENSE: MIT, copyright line exactly `Copyright (c) 2026 Saber5656` (accepted value;
   the implementation PR description must contain the line "LICENSE holder: Saber5656 —
   confirm or amend" so the owner can override).
10. One smoke test `tests/unit/smoke.test.ts` asserting `1 + 1 === 2` so the vitest wiring
    is proven.

## Acceptance Criteria

- [ ] `npm ci && npm run lint && npm run typecheck && npm run test && npm run build` all
      succeed locally on Node 20.
- [ ] `node dist/cli.js` prints the placeholder line, exit 0.
- [ ] CI workflow runs the same five steps plus actionlint on push(main)/PR, matrix node
      20+22, actions pinned by SHA, workflow-level `permissions: contents: read`.
- [ ] No runtime dependencies present in `package.json`.
- [ ] Directory skeleton contains exactly: `src/cli/commands`, `src/config`, `src/core`,
      `src/formats`, `src/providers`, `src/validate`, `src/glossary`, `src/git`,
      `src/report`, `src/util`, `tests/{unit,integration,e2e,fixtures}`,
      `examples/workflows`, `schemas/` (checklist in PR).

## Validation

- Run the command chain in Acceptance Criteria on a clean checkout.
- Open a draft PR and confirm CI is green on both Node versions.
- `npm pack --dry-run` lists only `dist/`, `schemas/`, `package.json`, README, LICENSE.

## Dependencies

None (first issue).

## Non-goals

Product logic, config schema, release/publish automation, README content beyond a stub
line, monorepo tooling (rejected by ADR-001).

## Design References

- DESIGN §5 (layout, dependency allowlist), §17 (CI matrix, static checks), §16 T-SUPPLY
- ADR-001 (single package decision)
