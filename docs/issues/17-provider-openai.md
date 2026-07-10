# Title

OpenAI provider

## Summary

Implement the `openai` TranslationProvider using the official `openai` SDK: chat request
built from the shared prompt, JSON-object response mode, `baseUrl` override support, and
mocked contract tests.

## Context

First real provider; also serves OpenAI-compatible endpoints (Azure, self-hosted
gateways) through `baseUrl` (DESIGN §10.2). Must be a thin, stateless HTTP client — all
resilience lives in Issue 16.

## Scope

- In: `src/providers/openai.ts`, registry wiring, mocked tests.
- Out: retries/backoff (Issue 16), prompt text (Issue 15), streaming.

## Detailed Requirements

1. Add runtime dep `openai` (allowlisted). Construct client with
   `{ apiKey: init.apiKey, baseURL: init.baseUrl ?? undefined, timeout:
   init.requestTimeoutMs, maxRetries: 0 }` — **SDK retries disabled** (Issue 16 owns
   retries; double-retry is a bug).
2. `translateBatch`: `buildPrompt(req)` → chat completions request
   `{ model: req.model, messages: [{role:"system",…},{role:"user",…}],
   response_format: { type: "json_object" }, temperature: 0 }`, pass through
   `signal`. Delegation contract (uniform across providers): the provider hands the raw
   response text to `parseProviderResponse` and returns its `translations` as-is —
   extra-id dropping, missing-id reporting, and corrective retries are the shared
   layer's job; the provider never synthesizes, fills, or filters translations itself.
   Map `usage.prompt_tokens/completion_tokens` → `inputTokens/outputTokens`.
3. Error mapping: SDK/HTTP errors → `ProviderError` with `retryable` true for
   network/5xx/429, false for 400/401/403/404; message includes status + provider name
   but NEVER request contents or headers (T-SECRET; redactor is backstop, not excuse).
4. Empty `choices`/null content → `ProviderError(retryable: true)`.
5. Tests (nock or the SDK's fetch injection; no live API), in
   `tests/unit/providers/openai.test.ts`:
   - request assertion: URL (default and overridden baseUrl — the override arrives
     pre-validated by the registry's `assertSafeBaseUrl`, Issue 14; this provider only
     passes it through, asserted by the mock host), auth header uses the init key, body
     contains `response_format: json_object`, temperature 0, both messages;
   - happy path parses translations + usage;
   - 429/500 → `ProviderError(retryable: true)`; 401 → non-retryable;
   - abort via signal → rejects promptly (fake timers);
   - fenced-JSON content → accepted via the shared parser's fence tolerance
     (delegation test);
   - one integration case: provider driven through Issue 16's `runBatches` against the
     mock (same file).
6. Provider must not read env directly — credentials arrive via `ProviderInit` from the
   registry resolver (DESIGN §10.2); test by constructing with explicit init while env
   is unset, plus the Issue 14 source-text check (`process.env` absent from this
   module). Missing-key `EnvError` behavior is the resolver's contract (tested in 14),
   not this module's.

## Acceptance Criteria

- [ ] SDK retries disabled; exactly one HTTP call per `translateBatch` (mock counts).
- [ ] Error mapping table above fully tested.
- [ ] `baseUrl` override hits the alternate host (mock asserts); no URL/protocol
      validation logic duplicated here (registry owns it).
- [ ] No env access inside the module; key never in any thrown message (test greps).
- [ ] The named Issue-16 integration case passes in
      `tests/unit/providers/openai.test.ts`.

## Validation

- `npx vitest run tests/unit/providers/openai.test.ts` green, offline.

## Dependencies

14 (registry/resolver/`assertSafeBaseUrl`; `ProviderError` semantics arrive via 14's
re-export of Issue 03 classes), 15, 16

## Non-goals

Model selection logic/defaults (config owns `model`), Azure auth schemes beyond
key+baseUrl, streaming, tool calling.

## Design References

- DESIGN §10.1–10.4, §16 T-SECRET/T-NET
- ADR-003
