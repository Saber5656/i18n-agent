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
   computeDiff(input: { source: Catalog; targets: Map<locale, Catalog>;
     lockfile: Lockfile; targetLocales: string[] }): DiffResult
   ```
2. Classification implements DESIGN §9.2 rows 1–6 exactly, using `hashValue` from
   Issue 04. Verbatim entries (meta flag from Issue 04): class `copiedVerbatim` when the
   target value differs or is absent; `fresh`-equivalent (not listed) when identical.
3. Missing target file (no catalog) ⇒ every source key is `missing` for that locale.
4. Deterministic ordering: items sorted by `(fileId, source-catalog key order, locale)` —
   the source catalog's insertion order is the canonical order (drives append order and
   stable reports).
5. Entries whose source value exceeds caps were already rejected by adapters; diff does
   no re-validation (single-responsibility note in code).
6. Tests: table test with one case per DESIGN §9.2 row; bootstrap scenario (empty
   lockfile → adopted+missing only, zero stale); staleness triggered by exact one-char
   source edit; NFC/NFD equivalence does NOT create stale (hash normalization); orphan
   detection with prune flag downstream (flag lives in pipeline — diff only classifies);
   deterministic order (shuffled input maps → identical output order); multi-locale
   matrix (2 files × 3 locales).

## Acceptance Criteria

- [ ] Every row of DESIGN §9.2 has a named test (`row1_missing`, … `row6_verbatim`).
- [ ] Bootstrap produces zero `stale` and zero provider work for already-translated keys.
- [ ] NFD-encoded source equal to NFC lockfile hash is `fresh`, not `stale`.
- [ ] Output order is deterministic under input permutation (test).
- [ ] Function is pure: no IO, no globals, no Date/random.

## Validation

- `npx vitest run tests/unit/core/diff.test.ts` green.
- Mutation spot-check: flipping any classification condition breaks at least one test
  (implementer runs a manual sanity mutation on rows 2/4).

## Dependencies

04, 11

## Non-goals

Reading files, writing lockfile updates, cost estimation, locale fallback chains
(e.g. `pt` ← `pt-BR` inheritance is NOT in v1).

## Design References

- DESIGN §9.2 (decision table), §9.3 (transitions), §7 (verbatim)
