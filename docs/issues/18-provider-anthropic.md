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
3. Response: concatenate `text`-type content blocks in array order, ignoring non-text
   blocks; zero text blocks or empty concatenation → `ProviderError(retryable: true,
   "empty-response")`. Result text → `parseProviderResponse` (same delegation contract
   as Issue 17: provider never synthesizes/filters translations); usage maps
   `input_tokens/output_tokens`.
4. Error mapping table (normative for this module):

   | Condition | ProviderError.retryable |
   |---|---|
   | network error / abort-by-timeout | true |
   | HTTP 429, 500–504, 529, or SDK `overloaded_error` type | true |
   | HTTP 400 / 401 / 403 / 404 | false |
   | `stop_reason: "max_tokens"` (truncated) | false, reason `response-truncated` |
   | zero/empty text blocks | true, reason `empty-response` |

   Messages include status + provider name, never request contents or headers.
5. Tests (mocked transport, offline) in `tests/unit/providers/anthropic.test.ts`:
   request assertions (endpoint incl. pre-validated baseUrl override passthrough,
   `x-api-key` header, system + single user message, temperature 0, max_tokens formula
   at boundary values); happy path + usage; every row of the mapping table; abort;
   non-text/multi-block response handling; one integration case through Issue 16's
   `runBatches` against the mock.
6. No env access; init-only credentials (same test pattern + source-text check as
   Issue 17).

## Acceptance Criteria

- [ ] Every row of the requirement-4 mapping table has a dedicated test.
- [ ] Request-shape assertions (endpoint/header/messages/temperature/max_tokens
      boundaries), usage mapping, abort, block-handling, and the Issue-16 integration
      case all pass (checklist mirrors requirement 5 — self-contained, no cross-issue
      reference needed).
- [ ] `max_tokens` formula implemented and boundary-tested; truncation surfaces as
      non-retryable with actionable message.
- [ ] SDK retries disabled; one HTTP call per translateBatch.
- [ ] Key never observable in errors/serialization; no `process.env` in module.

## Validation

- `npx vitest run tests/unit/providers/anthropic.test.ts` green, offline.

## Dependencies

14, 15, 16

## Non-goals

Tool-use/structured-output APIs (text JSON contract is the v1 baseline), prompt caching,
streaming, model defaults.

## Design References

- DESIGN §10.1–10.4, §16 T-SECRET, T-NET (baseUrl arrives pre-validated; this module
  must not weaken it)
- ADR-003
