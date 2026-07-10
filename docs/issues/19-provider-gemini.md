# Title

Google Gemini provider

## Summary

Implement the `gemini` TranslationProvider using `@google/genai`: generateContent with
system instruction + JSON response mime type, and the standard mocked contract tests.

## Context

Third provider; identical thin-client rules. Gemini offers a free tier, lowering the
trial barrier (Round 2 requirement).

## Scope

- In: `src/providers/gemini.ts`, registry wiring, mocked tests.
- Out: retries (Issue 16), prompt text (Issue 15), Vertex AI auth.

## Detailed Requirements

1. Add runtime dep `@google/genai`. Initialize with `{ apiKey: init.apiKey }`;
   honor `init.baseUrl` via the SDK's endpoint/httpOptions override if supported —
   otherwise implement the call with `fetch` keeping the SDK for types only
   (implementer picks ONE approach and documents it in code). Normative REST contract
   for the fetch path:
   `POST <base>/v1beta/models/<encodeURIComponent(model)>:generateContent` where
   `<base>` = `init.baseUrl ?? "https://generativelanguage.googleapis.com"`, body
   `{ systemInstruction: { parts: [{ text: prompt.system }] },
      contents: [{ role: "user", parts: [{ text: prompt.user }] }],
      generationConfig: { temperature: 0, responseMimeType: "application/json" } }`.
   `init.baseUrl` arrives pre-validated by the registry's `assertSafeBaseUrl` (Issue
   14, T-NET); this module only passes it through — no scheme logic here.
2. Request: abort via `signal`, timeout via init (no SDK auto-retries — disable or
   bypass; asserted by mock call count).
3. Response extraction rules (normative): no candidates AND
   `promptFeedback.blockReason` present → `ProviderError(retryable: false,
   "response-blocked: <blockReason>")`; candidate with
   `finishReason: "SAFETY"|"RECITATION"` → same non-retryable error naming the reason;
   candidate present but `content.parts` missing/empty → `ProviderError(retryable:
   true, "empty-response")`; multiple text parts → concatenate in order. Result text →
   `parseProviderResponse` (same delegation contract as Issues 17/18); usage from
   `usageMetadata.promptTokenCount/candidatesTokenCount` when present.
4. Error mapping policy identical to Issues 17/18 (retryable network/5xx/429;
   non-retryable 400/401/403/404). API key must be sent via header
   (`x-goog-api-key`) — never as a URL query parameter (keys in URLs leak into logs;
   T-SECRET). If the SDK insists on query params, use the fetch path.
5. Tests (offline, mocked HTTP) in `tests/unit/providers/gemini.test.ts`: endpoint +
   header auth assertion (explicitly assert the key is NOT in the URL), responseMimeType
   present, temperature 0; happy path + usage; every extraction rule of requirement 3
   (blocked, safety finish, missing parts, multi-part); error mapping
   (429/5xx retryable, 400/401/403/404 non-retryable); abort; one integration case:
   provider through Issue 16's `runBatches` asserting the provider itself performs no
   internal retry (mock call count = 1 per batch attempt; backoff belongs to the
   wrapper).
6. No env access; init-only credentials (+ source-text check as in Issue 17).

## Acceptance Criteria

- [ ] Auth travels in a header; test asserts the URL contains no key material.
- [ ] Safety-blocked responses produce a clear non-retryable ProviderError naming the
      finish reason.
- [ ] Standard contract test matrix (mirroring 17/18) passes offline.
- [ ] Exactly one HTTP call per translateBatch (no hidden SDK retries).

## Validation

- `npx vitest run tests/unit/providers/gemini.test.ts` green, offline.

## Dependencies

14, 15, 16

## Non-goals

Vertex AI / OAuth service accounts, multimodal input, streaming, model defaults.

## Design References

- DESIGN §10.1–10.4, §16 T-SECRET/T-NET
- ADR-003
