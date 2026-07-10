# Title

Provider interface, registry, and fake provider

## Summary

Define the `TranslationProvider` contract, credential resolution rules, the static
provider registry, and the deterministic `fake` provider used by tests and key-less runs.

## Context

Four real providers (Issues 17–20) implement this contract; batching/retry (Issue 16)
wraps it. Getting the contract, credential handling (ADR-003), and the fake provider
right first makes the provider issues mechanical.

## Scope

- In: `src/providers/types.ts`, `src/providers/registry.ts`, `src/providers/fake.ts`,
  credential resolution helper, unit tests.
- Out: prompt construction (Issue 15), batching/retry (Issue 16), real providers (17–20).

## Detailed Requirements

1. `types.ts` verbatim from DESIGN §10.1 (`TranslationProvider`, `TranslationItem`,
   `TranslationRequest`, `TranslationResult`) plus:
   ```ts
   export type ProviderName = "openai" | "anthropic" | "gemini" | "ollama" | "fake";
   export interface GlossaryTerm { source: string;
     translations: Record<string, string>; note?: string }
   export interface ProviderInit { name: ProviderName; model: string;
     baseUrl?: string; requestTimeoutMs: number; apiKey?: string }
   ```
   `GlossaryTerm` lives HERE (loader comes in Issue 21) so Issue 15 can proceed in
   parallel.
2. Credential resolution `resolveCredentials(cfg: Config["provider"]): ProviderInit` —
   the **single place** env is read (DESIGN §10.2): default `apiKeyEnv` per the §10.2
   table; ollama/fake require none; missing required env → `EnvError` naming the exact
   variable; on success it calls `registerSecretValue(key, apiKeyEnv)` (Issue 03) before
   returning. The resolved key is carried only inside `ProviderInit.apiKey`; providers
   store it in a private field (`#apiKey`) and never read `process.env` — enforced by a
   test asserting `JSON.stringify(provider)` contains no key material plus a source-text
   test asserting `process.env` does not appear in any concrete provider module.
3. `registry.ts`: static map `ProviderName → (init: ProviderInit) => TranslationProvider`
   (direct imports, ADR-005). `createProvider(cfg)` composes:
   `resolveCredentials(cfg)` → `assertSafeBaseUrl(init)` → constructor.
   `assertSafeBaseUrl` (exported, DESIGN §10.2/§16 T-NET defense-in-depth): `baseUrl`
   absent OR protocol `https:` OR hostname ∈ {`localhost`, `127.0.0.1`, `::1`} OR
   `allowInsecureBaseUrl === true`; otherwise `ConfigError`. Real providers register
   stubs throwing `ProviderError("not implemented")` (imported from `util/errors.ts`;
   test asserts the class and exit-code mapping 5) until their issues land.
4. `fake.ts`: deterministic, offline. For each item returns
   `«<targetLocale>» <sourceText>` (DESIGN §10.1) — placeholders preserved trivially
   because the source text is embedded verbatim. Special behaviors for tests, keyed by
   magic source strings: text containing `[[FAKE:FAIL]]` → id omitted from response;
   `[[FAKE:EXTRA]]` → response gains an unknown id (exercises id-allowlist handling in
   Issue 16); `[[FAKE:SLOW:<ms>]]` → delayed resolution where `<ms>` must match
   `^[0-9]{1,4}$` and is clamped to ≤ 5000 — a malformed marker is treated as literal
   text (no delay). Usage numbers: `inputTokens = Σ source chars`, `outputTokens =
   Σ output chars` (deterministic). E2E call-counter hook: when env
   `I18N_AGENT_FAKE_COUNTER` is set to a file path, each `translateBatch` call appends
   one line to that file (enables cross-process zero-provider-call assertions in
   Issue 32).
5. Timeout plumbing: providers receive `AbortSignal`; contract states they must abort
   promptly and surface `ProviderError` on abort (fake implements it; tested with fake
   timers).
6. Tests: registry returns constructors for all five names; credential defaults per
   provider table; `EnvError` names the variable; `registerSecretValue` called with the
   resolved key (spy); `assertSafeBaseUrl` matrix (https ok, http+localhost/127.0.0.1/
   ::1 ok, http+remote rejected, override flag accepted); stub classes are
   `ProviderError` with exit code 5; fake determinism (two runs identical), magic
   behaviors incl. malformed `[[FAKE:SLOW:x]]`, counter hook, abort behavior, no key in
   JSON serialization of any constructed provider (fake given a dummy key).

## Acceptance Criteria

- [ ] Types compile verbatim per DESIGN §10.1; `GlossaryTerm` exported from providers
      package.
- [ ] `resolveCredentials` implements the §10.2 table exactly (test per provider) and
      registers the key with the redactor (spy test).
- [ ] `assertSafeBaseUrl` enforces the exact T-NET host/protocol matrix.
- [ ] Fake provider is deterministic, offline, supports the three magic behaviors plus
      the counter hook, and respects AbortSignal.
- [ ] No dynamic imports in `src/providers/`; stubs present for unimplemented providers
      and typed as `ProviderError`.
- [ ] Key material never observable via `JSON.stringify` of any provider instance, and
      no provider module contains `process.env` (source-text test). End-to-end log
      redaction is covered by Issue 03's redactor tests and re-audited in Issue 36.

## Validation

- `npx vitest run tests/unit/providers/{registry,fake,credentials}.test.ts` green.
- `grep -R "apiKey" src/providers | grep -v "#apiKey\|ProviderInit\|apiKeyEnv"` reviewed
  manually — no stray copies.

## Dependencies

02, 03

## Non-goals

HTTP calls, prompt text, retries/batching, streaming, token pricing.

## Design References

- DESIGN §10.1–10.2, §16 T-SECRET, T-NET
- ADR-003, ADR-005
