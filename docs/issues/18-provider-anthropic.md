# Title

Anthropic provider

## Summary

Implement the `anthropic` TranslationProvider using `@anthropic-ai/sdk`: messages request
from the shared prompt, JSON-only output contract, and mocked contract tests mirroring
the OpenAI provider suite.

## Context

Second real provider — proves the abstraction (ADR-003). Same thin-client rules as
Issue 17.

## Scope

- In: `src/providers/anthropic.ts`, registry wiring, mocked tests.
- Out: retries (Issue 16), prompt text (Issue 15).

## Detailed Requirements

1. Add runtime dep `@anthropic-ai/sdk`. Client init
   `{ apiKey: init.apiKey, baseURL: init.baseUrl ?? undefined, timeout:
   init.requestTimeoutMs, maxRetries: 0 }` (SDK retries off).
2. `translateBatch`: `messages.create` with `{ model: req.model, system: prompt.system,
   messages: [{ role: "user", content: prompt.user }], max_tokens: computed, temperature: 0 }`,
   `signal` passed through. `max_tokens` heuristic: `min(8192, 512 + 4 × Σ source chars)`
   — document the constant; truncated responses (`stop_reason: "max_tokens"`) →
   `ProviderError(retryable: false, reason "response-truncated")` so Issue 16 splits are
   revisited by the operator rather than silently retried.
3. Response: concatenate `content[].text` blocks → `parseProviderResponse`; usage maps
   `input_tokens/output_tokens`.
4. Error mapping identical policy to Issue 17 (retryable: network/5xx/429/`overloaded`;
   non-retryable: 400/401/403/404), same no-secrets-in-messages rule.
5. Tests (mocked transport, offline): request assertions (endpoint incl. baseUrl
   override, `x-api-key` header, system + single user message, temperature 0, max_tokens
   formula at boundary values); happy path + usage; error mapping; abort; truncation →
   non-retryable error; integration case through Issue 16 runner.
6. No env access; init-only credentials (same test pattern as Issue 17).

## Acceptance Criteria

- [ ] Mirrors the Issue 17 test matrix 1:1 (same behaviors proven for this SDK).
- [ ] `max_tokens` formula implemented and boundary-tested; truncation surfaces as
      non-retryable with actionable message.
- [ ] SDK retries disabled; one HTTP call per translateBatch.
- [ ] Key never observable in errors/serialization.

## Validation

- `npx vitest run tests/unit/providers/anthropic.test.ts` green, offline.

## Dependencies

14, 15, 16

## Non-goals

Tool-use/structured-output APIs (text JSON contract is the v1 baseline), prompt caching,
streaming, model defaults.

## Design References

- DESIGN §10.1–10.4, §16 T-SECRET
- ADR-003
