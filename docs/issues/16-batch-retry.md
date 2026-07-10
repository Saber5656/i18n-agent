# Title

Batching, retry, and concurrency orchestration

## Summary

Implement `providers/batch.ts`: split pending diff items into provider batches, run them
with bounded concurrency, apply the uniform retry/backoff policy, isolate per-item
failures, and aggregate usage.

## Context

Resilience lives outside providers so all providers behave identically (DESIGN §10.4,
ADR-003). This module is the only place that decides when the provider is called again —
correctness here bounds cost (T-LOOP adjacent) and reliability.

## Scope

- In: `src/providers/batch.ts`, unit tests with fake timers and the fake provider.
- Out: prompt text (Issue 15), diff computation, file writes.

## Detailed Requirements

1. API:
   ```ts
   export interface BatchRunInput { provider: TranslationProvider;
     pending: DiffItem[];                       // missing ∪ stale from Issue 12
     cfg: { batchSize: number; maxConcurrency: number; maxRetries: number;
            requestTimeoutMs: number };
     shared: { sourceLocale: string; model: string; glossary: GlossaryTerm[];
               styleGuide?: string; projectContext?: string } }
   export interface BatchRunResult {
     translations: Map<string /* DiffItem id: fileId:locale:key */, string>;
     failures: Array<{ id: string; reason: string }>;
     usage: { inputTokens: number; outputTokens: number; requests: number } }
   runBatches(input: BatchRunInput): Promise<BatchRunResult>
   ```
2. Grouping (DESIGN §10.4): group `pending` by `targetLocale` (mixing fileIds); within a
   locale, chunk by `batchSize` items AND ≤ 8 000 chars cumulative source text —
   whichever limit hits first; single items > 8 000 chars go alone (adapter caps already
   bound them at 10 000).
3. Item ids: `<fileId>:<targetLocale>:<flatKey>` (colon-joined; flatKey may contain
   colons? — flatKey charset is unrestricted, so use `encodeURIComponent` on each part
   and document the encoding; tests must cover a key containing `:`).
4. Concurrency: `p-limit(maxConcurrency)` across ALL batches (not per locale).
   Each request wrapped in `AbortSignal.timeout(requestTimeoutMs)`.
5. Retry policy per request (DESIGN §10.4): retry on network error, HTTP 5xx, 429
   (provider errors must carry `retryable: boolean` — extend `ProviderError` with it,
   coordinating with Issue 03's class): backoff `min(1000·2^attempt + jitter(0..250),
   30000)` ms, at most `maxRetries` retries. Non-retryable (4xx auth/permission) →
   `ProviderError` fails the whole run fast (exit 5 upstream).
6. Content problems (from Issue 15's `parseProviderResponse`): `not-json`/prose → ONE
   corrective re-request of the same batch with an appended system line "Previous reply
   was not valid JSON; reply with only the JSON object." Missing ids after that → each
   missing id retried once in a single-item batch; still missing → failure
   `reason: "missing-translation"`. Extra ids/non-strings are dropped and logged (never
   written).
7. Failure isolation: item failures never abort the run; aggregate into `failures` for
   exit code 2 upstream. If EVERY batch fails with retryable exhaustion →
   `ProviderError` (exit 5).
8. Usage aggregation sums provider-reported usage when present; `requests` counts actual
   HTTP-level attempts (including retries).
9. Determinism aid: batches are formed in input order; jitter comes from an injectable
   `rng: () => number` (default `Math.random`) so tests pass a seeded stub.
10. Tests (fake timers + fake provider + a scripted mock provider): chunking by count and
    by char budget; a key containing `:` and unicode; 429 → backoff schedule asserted
    (timer values); auth 4xx → fast `ProviderError`; `[[FAKE:FAIL]]` → single-item retry
    then failure recorded; `[[FAKE:EXTRA]]` → extra id dropped, logged; timeout → abort
    surfaced as retryable; concurrency ceiling observed (max in-flight counter);
    all-batches-fail → ProviderError.

## Acceptance Criteria

- [ ] Chunking respects both caps; boundary tests at exactly `batchSize` and 8 000 chars.
- [ ] Backoff sequence matches the formula (asserted via fake timers, seeded jitter).
- [ ] Auth failures abort fast; item content failures never abort the run.
- [ ] Ids are collision-safe for arbitrary flatKeys (encoding tested).
- [ ] In-flight requests never exceed `maxConcurrency` (instrumented test).

## Validation

- `npx vitest run tests/unit/providers/batch.test.ts` green (runs with fake timers,
  < 5 s wall clock).

## Dependencies

14. (Uses Issue 12's `DiffItem` type; both are Wave ≤ 3 — import type only.)

## Non-goals

Streaming, rate-limit header parsing (v2 refinement), cross-run caching (v2), cost
estimation in currency.

## Design References

- DESIGN §10.4, §13 (exit 2/5 semantics), §10.1 (result contract)
