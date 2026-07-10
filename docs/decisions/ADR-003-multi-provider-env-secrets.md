# ADR-003: Multi-provider LLM abstraction with env-only BYO credentials

- Status: Accepted (2026-07-10)
- Deciders: product owner (Round 1/2: multi-provider, OpenAI+Anthropic+Gemini+Ollama), Fable

## Context

Translation quality/cost/access differ per organization; an OSS tool must stay
provider-neutral. Credentials handling is the most security-sensitive part of the design
and must be uniform across shells, CI, and the Action.

## Decision

1. A minimal **`TranslationProvider` interface** (DESIGN §10.1) with implementations for
   OpenAI, Anthropic, Google Gemini, Ollama, and a deterministic `fake` provider; batching,
   retry, and concurrency live **outside** providers so resilience is uniform.
2. **API keys exist only in environment variables.** Config stores the env var *name*
   (`apiKeyEnv`), never a value. Keys are resolved inside provider constructors, held
   privately, and scrubbed from logs/errors by the global redactor (DESIGN §16 T-SECRET).
3. `baseUrl` overrides must be https unless localhost or `allowInsecureBaseUrl: true`
   (T-NET), which also cleanly serves Azure-OpenAI-compatible and self-hosted endpoints.

## Alternatives considered

- **Single provider first** — thinnest v1 but lock-in, weak OSS appeal; rejected by owner.
- **Keys in config file** — one file to leak; encourages committing secrets; rejected.
- **OS keychain integration** — desktop-only, useless in CI; deferred indefinitely.
- **Generic engine plugins incl. DeepL in v1** — broader abstraction cost now for demand
  we have not validated; interface is shaped so a DeepL adapter is additive in v2.

## Consequences

- Two+ real providers force the abstraction to be honest from day one; `fake` gives
  hermetic tests and key-less pipeline runs.
- Every provider issue (17–20) is small and mechanical: implement one HTTP client against
  a fixed contract with mocked tests.
- Users own provider choice, cost, and data-privacy tradeoffs; README must state plainly
  that source strings are sent to the configured provider.
