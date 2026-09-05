# Title

FormatAdapter interface, registry, and conformance harness

## Summary

Define the `FormatAdapter` interface and static registry, plus a reusable conformance
test harness that every concrete adapter (Issues 06–10) must pass unchanged.

## Context

Five formats must behave identically at the contract level (minimal-churn writes, append
ordering, orphan retention, boilerplate synthesis — DESIGN §8.1–8.2). Encoding the
contract as a shared executable spec prevents five divergent interpretations by five
implementation runs.

## Scope

- In: `src/formats/types.ts`, `src/formats/registry.ts`, placeholder profile id type,
  `tests/formats/conformance.ts` (exported harness), harness self-test with an in-memory
  reference adapter.
- Out: concrete adapters (06–10), placeholder extraction logic (Issue 22).

## Detailed Requirements

1. `types.ts` exactly per DESIGN §8.1:
   ```ts
   export type FormatId = "json" | "yaml" | "arb" | "ios-strings" | "android-xml";
   export type PlaceholderProfileId =
     "icu" | "i18next" | "rails-percent" | "printf-ios" | "printf-java" | "tags";
   export interface ParseInput { raw: string; fileId: string; locale: string;
     options: Record<string, unknown> }
   export interface SerializeInput { existingRaw: string | null; catalog: Catalog;
     sourceCatalog: Catalog; prune: boolean; options: Record<string, unknown> }
   export interface FormatAdapter {
     readonly format: FormatId;
     readonly defaultPlaceholderProfiles: PlaceholderProfileId[];
     parse(input: ParseInput): Catalog;
     serialize(input: SerializeInput): string;
   }
   ```
   `existingRaw: null` ⇒ synthesize a new file incl. format boilerplate (DESIGN §8.2).
   Also export `emptyCatalog(fileId: string, locale: string): Catalog` — the
   representation of a missing target file; callers never invoke `parse` for absent
   files (DESIGN §8.2).
2. `registry.ts`: `const adapters: Record<FormatId, FormatAdapter>` — a static map
   populated by direct imports (ADR-005). `getAdapter(format: string)` (accepts `string`
   so the unknown-id defense is testable) throws `FormatError` for unknown ids.
   Until Issues 06–10 land, entries may be `TODO` stubs throwing `FormatError("not
   implemented")` so the package always compiles.
3. Conformance harness `defineFormatConformance(opts: ConformanceSpec)` with an explicit
   fixture contract:
   ```ts
   export interface ConformanceSpec {
     adapter: FormatAdapter;
     fixtures: ConformanceFixture[];
   }
   export interface ConformanceFixture {
     name: string;
     raw: string;                    // file content (or path under tests/fixtures/)
     locale: string; fileId: string;
     options?: Record<string, unknown>;
     stable: boolean;                // participates in byte round-trip check
     churnBudgetLines?: number;      // max changed lines for a one-value edit (default 3)
     editKey?: string;               // flatKey targeted by the churn test
     sourceForAppend?: CatalogEntry[]; // entries appended in the append-order test
   }
   ```
   The harness generates `describe` blocks asserting, at minimum:
   - parse→serialize with unchanged catalog is byte-identical (round-trip stability) for
     the adapter's `stable` fixtures;
   - updating one existing value changes only that value's line(s) (churn check via
     line-diff count ≤ fixture-declared budget);
   - new keys are appended at the end of their parent container in source-catalog order;
   - orphan entries survive `prune: false` and are removed with `prune: true`;
   - `existingRaw: null` produces a file that re-parses to the same catalog and contains
     the format's boilerplate;
   - resource limits from Issue 04 are enforced (oversized fixture rejected);
   - every entry's `keyPath`/`value` survives parse(serialize(parse(x))) (idempotence);
   - serialized output ends with exactly one trailing `\n` (not zero, not two).
4. Harness self-test (`tests/formats/conformance.selftest.ts`): a reference JSON-lines
   adapter plus **eight deliberately-broken variants — one per contract rule above**
   (reorders keys, inflates churn, prepends instead of appends, drops orphans without
   prune, omits boilerplate, ignores limits, mutates values on round-trip, emits double
   trailing newline); the self-test asserts the harness fails each variant
   (`expect(...).rejects` / failing assertion captured via vitest `expect(fn).toThrow`).

## Acceptance Criteria

- [ ] Interface compiles and matches DESIGN §8.1 verbatim (field names/types), plus
      `emptyCatalog` export.
- [ ] Registry is static; no dynamic import/require anywhere in `src/formats/`.
- [ ] Harness covers the eight contract rules and the self-test proves each rule can
      fail via its dedicated broken adapter variant.
- [ ] Adapter issues 06–10 can adopt the harness by providing `ConformanceFixture[]`
      only (documented usage comment at top of `conformance.ts`).

## Validation

- `npx vitest run tests/formats` green (harness self-test).
- `grep -R -E "import\(|require\(" src/formats` returns nothing.

## Dependencies

03, 04

## Non-goals

Any real format parsing; placeholder profile *semantics* (only the id union lives here);
plugin loading (ADR-005).

## Design References

- DESIGN §8.1–8.2 (contract), §7 (limits), §17 (conformance layer)
- ADR-005
