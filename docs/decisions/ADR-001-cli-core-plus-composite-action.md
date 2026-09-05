# ADR-001: CLI core (single npm package) + composite GitHub Action wrapper

- Status: Accepted (2026-07-10)
- Deciders: product owner (Round 1 requirements confirmation), Fable (design)

## Context

i18n-agent must run both in developer shells and unattended in CI, and the README promise
("translates and opens a PR") implies first-class GitHub integration. We must pick the
delivery architecture and repository shape before any issue can be scoped.

## Decision

1. Implement all product behavior in **one npm package `i18n-agent`** exposing a CLI
   (TypeScript, ESM, Node >= 20). No monorepo in v1.
2. Ship GitHub integration as a **composite action** (`action.yml` at this repository's
   root) that runs `npx --yes i18n-agent@<EXACT_VERSION>`; the pinned version literal is
   bumped only by the release pipeline.
3. The Action adds **no behavior** beyond environment wiring (node setup, token env,
   argument pass-through). Every feature must be reachable from the bare CLI.

## Alternatives considered

- **JS action with bundled `dist/`** — faster cold start, but requires committing built
  artifacts and a second build pipeline; version skew between npm and action bundles is a
  real failure mode.
- **Monorepo (`packages/core|cli|action`)** — cleaner layering on paper, but tripled
  scaffolding and release complexity for a v1 with one publishable artifact.
- **GitHub App / hosted bot** — persistent credentials, hosting, and multi-tenant security
  for a v1 OSS project; rejected as disproportionate.

## Consequences

- One build, one test suite, one release line; issues can assume a single `src/` tree.
- Action runs pay an `npx` install cost per job (acceptable; documented).
- GitHub-specific code is isolated behind `git/githubPr.ts` so a future GitLab adapter or
  monorepo split does not ripple through core.
- The action version pin means a compromised/latest npm release cannot silently reach
  `@v1` action users (supply-chain containment, DESIGN §16 T-SUPPLY).
