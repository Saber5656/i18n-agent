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
   otherwise implement the call with `fetch` against
   `<baseUrl ?? https://generativelanguage.googleapis.com>` keeping the SDK for types
   only (implementer picks ONE approach and documents it in code; the request/response
   contract below is normative either way).
2. Request: model `req.model`, `systemInstruction` = prompt.system, single user content =
   prompt.user, generation config `{ temperature: 0, responseMimeType: "application/json" }`,
   abort via `signal`, timeout via init (no SDK auto-retries — disable or bypass).
3. Response: first candidate text → `parseProviderResponse`; usage from
   `usageMetadata.promptTokenCount/candidatesTokenCount` when present. Empty
   candidates / safety-blocked → `ProviderError(retryable: false,
   "response-blocked: <finishReason>")`.
4. Error mapping policy identical to Issues 17/18 (retryable network/5xx/429;
   non-retryable 400/401/403/404). API key must be sent via header
   (`x-goog-api-key`) — never as a URL query parameter (keys in URLs leak into logs;
   T-SECRET). If the SDK insists on query params, use the fetch path.
5. Tests (offline, mocked HTTP): endpoint + header auth assertion (explicitly assert the
   key is NOT in the URL), responseMimeType present, temperature 0; happy path + usage;
   blocked response → non-retryable; error mapping; abort; Issue 16 integration case.
6. No env access; init-only credentials.

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
