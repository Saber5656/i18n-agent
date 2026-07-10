# ADR-005: No dynamic plugin loading in v1 (static registries only)

- Status: Accepted (2026-07-10)
- Deciders: Fable (conservative security default; owner informed via DESIGN review)

## Context

Format adapters, providers, and validators are natural extension points, and "plugin
system" is a common OSS request. Dynamic loading (e.g. `plugins: ["./my-adapter.js"]` or
npm-resolved plugin names in config) means the config file — a semi-trusted, contributor-
editable input inside the repo — could cause arbitrary code execution in CI runners that
hold provider keys and a writable GitHub token.

## Decision

v1 ships **compile-time static registries only** (DESIGN §4, §8.1, §10.1): the sets
`formats = {json, yaml, arb, ios-strings, android-xml}`, `providers = {openai, anthropic,
gemini, ollama, fake}`, and the five validators are fixed at build time. Config may only
*select and parameterize* built-ins; it can never cause module resolution or code loading.
Extension in v1 = contribute an adapter upstream via PR.

## Alternatives considered

- **`import()` of config-referenced modules** — turns every locale-file PR reviewer into a
  security gate for RCE; unacceptable given T-INJ/T-FORK posture.
- **Plugins with allowlist/signature scheme** — real design work (trust roots, versioning)
  for zero confirmed demand; deferred to v2 with its own ADR if demand appears.

## Consequences

- The threat model for config stays "data only", keeping DESIGN §16 T-PATH/T-INJ tractable
  and Issue 02 mechanically testable.
- Interfaces are still designed as clean ports, so a future plugin ADR changes packaging,
  not architecture.
- Niche formats (PO, XLIFF, …) are roadmap items rather than user-side plugins in v1.
