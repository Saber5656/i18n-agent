# Title

Ollama provider (local LLM)

## Summary

Implement the `ollama` TranslationProvider against the local Ollama HTTP API
(`/api/chat`, `format: "json"`), with no API key, localhost default, and mocked tests.
This provider also unlocks fully offline E2E usage.

## Context

Key-less local translation was a Round 2 requirement (trialability, offline testing).
Quality is model-dependent and explicitly not guaranteed (DESIGN §10.2); the engineering
contract is identical to other providers.

## Scope

- In: `src/providers/ollama.ts`, registry wiring, mocked tests.
- Out: model management (`ollama pull`), retries (Issue 16), prompt text (Issue 15).

## Detailed Requirements

1. No SDK — plain `fetch` (Node 20 global; tests mock it via `vi.stubGlobal("fetch",…)`
   — no new dependency). Base URL: `init.baseUrl ?? "http://localhost:11434"`.
   T-NET enforcement lives in config (Issue 02) + registry `assertSafeBaseUrl`
   (Issue 14) with the exact allowed plain-http hosts `localhost`, `127.0.0.1`, `::1`;
   this module passes the pre-validated URL through and must not add its own host
   parsing (a passthrough test with each allowed host form is required).
2. Request: `POST <base>/api/chat` body
   `{ model: req.model, stream: false, format: "json", options: { temperature: 0 },
   messages: [{role:"system",content:prompt.system},{role:"user",content:prompt.user}] }`,
   `signal` passed to fetch, manual timeout via `AbortSignal.any([signal,
   AbortSignal.timeout(init.requestTimeoutMs)])`.
3. Response: `message.content` → `parseProviderResponse`; usage from
   `prompt_eval_count`/`eval_count` when present.
4. Error mapping: `ECONNREFUSED`/fetch network TypeError → `ProviderError(retryable:
   true)` — consistent with DESIGN §10.4's "network errors retry" — whose message is
   the actionable "Ollama not reachable at <base> — is `ollama serve` running?" (the
   hint therefore appears in the final failure after Issue 16 exhausts retries);
   HTTP 404 model missing → non-retryable naming the model and hinting `ollama pull`;
   5xx → retryable; malformed body → handled by shared parser + Issue 16 policy.
5. Tests (offline, `vi.stubGlobal("fetch", …)`) in
   `tests/unit/providers/ollama.test.ts`: request body assertions (format json, stream
   false, temperature 0); happy path + usage mapping; ECONNREFUSED → retryable with
   serve-hint message; 404 model hint; abort/timeout composition (fake timers);
   allowed-host passthrough forms; one Issue 16 integration case (same file).
6. Credential rules: this provider never reads env, never sends an `Authorization`
   header, and ignores `ProviderInit.apiKey` if present (tests assert no auth header on
   the wire and no `process.env` in module source).

## Acceptance Criteria

- [ ] Works offline in tests; failure messages are actionable (serve/pull hints tested);
      connection-refused is retryable per DESIGN §10.4.
- [ ] Timeout composition (caller signal + request timeout) proven with fake timers.
- [ ] All requirement-5 cases pass in `tests/unit/providers/ollama.test.ts`; one HTTP
      call per translateBatch.
- [ ] No auth header ever sent; no env reads (wire + source-text assertions).

## Validation

- `npx vitest run tests/unit/providers/ollama.test.ts` green, offline.
- Manual (optional, documented as such): live smoke against a local Ollama with any
  small model — translations return and parse.

## Dependencies

14, 15, 16

## Non-goals

Model download/management, GPU configuration, quality guarantees, streaming.

## Design References

- DESIGN §10.1–10.4, §16 T-NET
- ADR-003
