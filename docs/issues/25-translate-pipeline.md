# Title

Translation pipeline and `translate` command

## Summary

Implement `core/pipeline.ts` — the orchestration detect → breaker → translate → validate
→ write files → update lockfile — and the `i18n-agent translate` command with
`--dry-run`, `--prune`, `--allow-large`, `--reset-lockfile`, `--locale`, `--file`.

## Context

This is the product's beating heart, composing Waves 0–4 into the W3 local workflow and
the engine that `pr` (Issue 28) wraps (DESIGN §3.3, §13). The empty-diff short-circuit
here is a load-bearing loop/cost control (T-LOOP).

## Scope

- In: `src/core/pipeline.ts`, `src/cli/commands/translate.ts`, integration tests with the
  fake provider across all five formats.
- Out: git/PR operations (26–28), provider internals.

## Detailed Requirements

1. Pipeline API:
   ```ts
   export interface PipelineOptions { locales?: string[]; fileIds?: string[];
     dryRun: boolean; prune: boolean; allowLarge: boolean; resetLockfile: boolean }
   export interface PipelineResult { report: RunReport;
     wroteFiles: string[]; lockfileUpdated: boolean }
   runPipeline(loaded: LoadedConfig, opts: PipelineOptions): Promise<PipelineResult>
   ```
2. Ordered steps (each numbered step must be observable in `--verbose` logs):
   1. resolve paths (Issue 13 helper) with filters. Filter rules: unknown locale/file id
      → `UsageError` exit 3 listing configured values; source locale not filterable;
      repeated flags deduped;
   2. parse source+target catalogs;
   3. lockfile: `--reset-lockfile` → start from empty ledger (bootstrap semantics rebuild);
      else load;
   4. `computeDiff`;
   5. **full short-circuit**: `pending ∪ adoptions ∪ verbatimSync ∪ (prune ? orphans : ∅)
      = ∅` → return `outcome: "clean"` with no writes at all;
   6. cost breaker: `pending.length > maxKeysPerRun && !allowLarge` → `UsageError`
      (exit 3) with count + remediation (DESIGN §13);
   7. `--dry-run`: emit report of planned work incl. estimated request count via
      Issue 16's pure `planBatches` (same `batchSize` and 8 000-char cap as
      `runBatches`; batch-count boundary fixtures asserted), write NOTHING, exit 0;
   8. **provider gate**: a provider is constructed ONLY when `pending ≠ ∅`
      (spy-asserted). Adoption/verbatim/prune-only runs skip straight to steps 10–12
      and never touch provider code (bootstrap runs stay key-less and cost-free);
      when pending work exists: glossary/style load (Issue 21) → `runBatches`
      (Issue 16) with provider from registry (Issue 14);
   9. per translated item: `validateEntry` (Issue 24) — accepted → stage into target
      catalog + `recordTranslation`; rejected → failure list (lockfile untouched for
      that entry, DESIGN §11.1);
   10. adoptions → `adoptEntry`; verbatimSync → copy values; prune → delete orphans from
       target catalogs (and lockfile GC);
   11. serialize + write target files (only files whose catalog changed). Atomic write
       spec: write to `.<basename>.i18n-agent.tmp` in the same directory (paths already
       root-confined by Issue 02), then `rename` over the target; tmp files removed on
       failure (try/finally); no writes outside target catalogs + lockfile; write
       errors redacted like all errors. `saveLockfile` once at end; lockfile GC applied;
   12. `RunReport` (`command: "translate"`, outcome `applied`/`partial`/`clean`).
       Exit-code rule (DESIGN §10.4/§13): non-retryable provider auth failure or ALL
       batches failing → `ProviderError` exit 5 and NO file writes; per-item failures
       (missing ids, validation rejections) with ≥ 0 accepted items → writes proceed
       for accepted items, exit 2; no failures → exit 0.
3. Write discipline: never touch source files; never write on `--dry-run` or when any
   step before 9 fails; partial per-item failure still writes accepted entries
   (documented partial semantics, exit 2).
4. Integration tests (fake provider, all five adapters via fixtures): full happy path
   per format (assert file bytes + lockfile content); stale retranslation flow (edit
   source → stale → new value written, hash updated); rejection flow (fake returns
   broken placeholder via a scripted provider stub → entry NOT written, failure
   reported, exit 2); dry-run writes nothing (tree hash); breaker at boundary
   (`maxKeysPerRun`, `=`, `+1`, `--allow-large`); prune on/off; `--reset-lockfile`
   adopts current state; short-circuit run performs zero provider constructions (spy);
   adoption-only/bootstrap run writes lockfile but constructs no provider (spy);
   auth-failure run (scripted provider) exits 5 with zero file writes; unknown
   `--locale`/`--file` → exit 3; atomic write leaves no tmp files (incl. on injected
   write failure).
5. Concurrency note: pipeline itself is sequential per file group; provider concurrency
   is inside Issue 16 (document — no extra parallelism here in v1).

## Acceptance Criteria

- [ ] Step order, full short-circuit, and the pending-only provider gate proven by
      tests (provider spy on clean AND adoption-only runs).
- [ ] Rejected translations never reach disk or lockfile (fixture-proven).
- [ ] Exit-code rule: per-item failures → 2 with accepted writes; auth/all-batches
      failure → 5 with zero writes (both integration-tested).
- [ ] Dry-run and clean runs are write-free (tree-hash assertions); dry-run batch
      estimates equal `planBatches` output at the boundary fixtures.
- [ ] Works end-to-end for json, yaml, arb, ios-strings, android-xml fixtures with the
      fake provider.

## Validation

- `npx vitest run tests/integration/translate.test.ts` green (matrix over 5 formats).

## Dependencies

12, 13, 14 (provider registry/`createProvider`), 16, 21, 24 (+06–10 for the full format
matrix; runnable against 06 alone during development)

## Non-goals

Git operations, PR bodies, resume/checkpointing for huge backfills (DESIGN §19 U6),
translation caching (v2).

## Design References

- DESIGN §3.3 (W3), §9.2 (classes), §10.4 (batching), §11.1 (rejection), §13 (flags/exit
  codes), §16 T-LOOP
