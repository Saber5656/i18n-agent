# Title

Missing/stale/orphan diff engine

## Summary

Implement the pure diff engine that classifies every `(fileId, flatKey, targetLocale)`
into the six classes of DESIGN §9.2 and produces the work plan consumed by translate/check.

## Context

This is the core detection logic — the "新規UI文字列を検知" of the product one-liner.
It must implement the DESIGN §9.2 decision table exactly; every row is a contract.

## Scope

- In: `src/core/diff.ts`, unit tests covering the full decision table and transitions
  (DESIGN §9.3).
- Out: lockfile IO (Issue 11), reporting (Issue 13), translation (Issue 25).

## Detailed Requirements

1. API (pure, synchronous):
   ```ts
   export type DiffClass = "missing" | "stale" | "fresh" | "adopted" | "orphan" | "copiedVerbatim";
   export interface DiffItem { fileId: string; key: string; targetLocale: string;
     cls: DiffClass; sourceValue?: string; targetValue?: string; sourceHash?: string;
     description?: string }
   export interface DiffResult { items: DiffItem[];
     counts: Record<DiffClass, number>;
     pending: DiffItem[];          // missing ∪ stale (translation work)
     adoptions: DiffItem[];        // lockfile writes without translation
     verbatimSync: DiffItem[] }    // non-string leaves to copy
   computeDiff(input: {
     files: Array<{ fileId: string; source: Catalog;
                    targets: Map<string /* locale */, Catalog> }>;  // multi-file in one call
     lockfile: Lockfile; targetLocales: string[] }): DiffResult
   ```
2. Classification implements DESIGN §9.2 rows 1–6 exactly, using `hashValue` from
   Issue 04. Lockfile precedence: `L…locales[t]` counts as **present** only when the
   file bucket, key entry, AND locale entry all exist; absence at any level ⇒ row 4
   (`adopted`) when the key is in S ∩ T. Verbatim entries (`isVerbatim` from Issue 04):
   compared by raw lexical text (`entry.value`); class `copiedVerbatim` when the target
   raw text differs or the key is absent from the target; treated as `fresh` when
   identical. Verbatim entries never enter `pending`.
3. Missing target file ⇒ caller passes `emptyCatalog(fileId, locale)` (Issue 05);
   every source key is then `missing` for that locale.
4. Deterministic ordering: items sorted by `(files[] input order, source-catalog key
   order, targetLocales order)`; orphan items sort after all source-keyed items of the
   same `(fileId, locale)`, in the **target** catalog's insertion order.
5. Entries whose source value exceeds caps were already rejected by adapters; diff does
   no re-validation (single-responsibility note in code).
6. Tests: table test with one case per DESIGN §9.2 row; bootstrap scenario (empty
   lockfile → adopted+missing only, zero stale); staleness triggered by exact one-char
   source edit; NFC/NFD equivalence does NOT create stale (hash normalization); orphan
   detection with prune flag downstream (flag lives in pipeline — diff only classifies);
   deterministic order (shuffled input maps → identical output order); multi-locale
   matrix (2 files × 3 locales).

## Acceptance Criteria

- [ ] Every row of DESIGN §9.2 has a named test (`row1_missing`, … `row6_verbatim`),
      plus dedicated branch tests `row4_missing_file_bucket`, `row4_missing_key_entry`,
      and `row4_missing_locale_entry` for the partial-lockfile precedence rule.
- [ ] Bootstrap produces zero `stale` and zero provider work for already-translated keys.
- [ ] NFD-encoded source equal to NFC lockfile hash is `fresh`, not `stale`.
- [ ] Output order (incl. orphan placement) is deterministic under input permutation.
- [ ] Function is pure: no IO, no globals, no Date/random.

## Validation

- `npx vitest run tests/unit/core/diff.test.ts` green (the named row/branch tests above
  are the objective mutation guard).

## Dependencies

04, 11

## Non-goals

Reading files, writing lockfile updates, cost estimation, locale fallback chains
(e.g. `pt` ← `pt-BR` inheritance is NOT in v1).

## Design References

- DESIGN §9.2 (decision table), §9.3 (transitions), §7 (verbatim)
