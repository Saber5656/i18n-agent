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

1. No SDK — plain `fetch`. Base URL: `init.baseUrl ?? "http://localhost:11434"`
   (localhost http allowed by the T-NET rule; non-localhost http requires
   `allowInsecureBaseUrl`, enforced already by config — add a defensive re-check here).
2. Request: `POST <base>/api/chat` body
   `{ model: req.model, stream: false, format: "json", options: { temperature: 0 },
   messages: [{role:"system",content:prompt.system},{role:"user",content:prompt.user}] }`,
   `signal` passed to fetch, manual timeout via `AbortSignal.any([signal,
   AbortSignal.timeout(init.requestTimeoutMs)])`.
3. Response: `message.content` → `parseProviderResponse`; usage from
   `prompt_eval_count`/`eval_count` when present.
4. Error mapping: `ECONNREFUSED`/fetch TypeError → `ProviderError(retryable: false)`
   with actionable message "Ollama not reachable at <base> — is `ollama serve` running?";
   HTTP 404 model missing → non-retryable naming the model and hinting `ollama pull`;
   5xx → retryable; malformed body → handled by shared parser + Issue 16 policy.
5. Tests (mocked fetch/undici, offline): request body assertions (format json, stream
   false, temperature 0); happy path + usage mapping; ECONNREFUSED message; 404 model
   hint; abort/timeout; Issue 16 integration case; defensive baseUrl re-check
   (https-or-localhost).
6. No env access, no key handling (constructing with a key is ignored — documented).

## Acceptance Criteria

- [ ] Works offline in tests; failure messages are actionable (serve/pull hints tested).
- [ ] Timeout composition (caller signal + request timeout) proven with fake timers.
- [ ] Standard contract matrix passes; one HTTP call per translateBatch.
- [ ] Defensive URL scheme check present and tested.

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
