# Title

JSON (i18next family) format adapter

## Summary

Implement the `json` FormatAdapter for i18next/react-intl/vue-i18n style JSON catalogs:
nested or flat key styles, indentation preservation, minimal-churn serialization.

## Context

JSON is the primary v1 format (DESIGN §8.3) and the reference implementation other
adapters are compared against. It is also the adapter used by `check`'s first integration
tests (Issue 13).

## Scope

- In: `src/formats/json.ts`, registry wiring, fixtures + conformance run, unit tests for
  format-specific rules.
- Out: ARB (Issue 08 — separate adapter even though ARB is JSON), placeholder validation.

## Detailed Requirements

1. `options` (zod-validated inside the adapter, `.strict()`):
   `{ keyStyle?: "nested" | "flat" }`, default `"nested"`.
2. `parse`: `JSON.parse` the file (must be an object at root; arrays/scalars at root →
   `FormatError`). Nested mode: recurse objects into `keyPath` segments; arrays of
   strings → one entry per element with `[<index>]` segment (Issue 04); non-string leaves
   (number/boolean/null) → entry with `meta: { verbatim: true }` and `value` = JSON
   stringification. Flat mode: top-level keys are single segments verbatim (no dot
   splitting — dots inside flat keys stay literal; escaping handled by `flatKey`).
3. Indentation detection: inspect the first indented line; support 2/4 spaces or tab;
   default 2 when empty/new. Preserve a single trailing newline always.
4. `serialize` (minimal churn, DESIGN §8.1): rebuild from `existingRaw`'s structure —
   existing keys keep their positions; updated values replace in place; new keys append
   at the end of their parent object in source-catalog order, creating intermediate
   objects as needed; orphans kept unless `prune`; verbatim entries written back as their
   original JSON scalar. Output via a deterministic writer (stable key iteration), NOT
   `JSON.stringify` of a reordered object.
5. `existingRaw: null` → `{}`-rooted file built from catalog in source order with detected
   default indent.
6. `defaultPlaceholderProfiles: ["icu", "i18next", "tags"]` (DESIGN §8.3).
7. Resource limits enforced via Issue 04 guards before parsing (bytes) and after (keys,
   value length).
8. i18next plural suffix keys (`_one`, `_other`, `_zero`, `_two`, `_few`, `_many`) are
   ordinary keys — assert one fixture covers them and document the CLDR caveat in code
   comment referencing DESIGN §2.2.
9. Fixtures (under `tests/fixtures/json/`): nested 2-space, nested 4-space, flat style,
   arrays, non-string leaves, plural suffixes, unicode/emoji values, a 5 MiB-exceeding
   generated case (constructed in test, not committed), duplicate-key JSON (last-wins per
   JSON.parse — document).
10. Register the conformance harness (Issue 05) with `stable` fixtures + churn budgets.

## Acceptance Criteria

- [ ] Conformance harness passes for all fixtures.
- [ ] Editing one value in a 100-key nested fixture changes exactly the lines of that
      value (churn budget test).
- [ ] New keys land at parent-end in source order; deep new paths auto-create objects.
- [ ] Flat mode never splits dotted keys; nested mode round-trips `"a.b"` object keys via
      escaping.
- [ ] Verbatim (non-string) leaves are byte-identical after round-trip and are reported
      distinctly (meta flag present).
- [ ] Indentation (2/4/tab) and trailing newline preserved per fixture.

## Validation

- `npx vitest run tests/unit/formats/json.test.ts tests/formats` green.
- Manual: run adapter against a real-world i18next `en.json` (≥ 200 keys) copied into a
  scratch dir; diff after a one-value change is one hunk.

## Dependencies

05

## Non-goals

JSON5/JSONC comments (not in v1), key sorting/canonicalization, plural-category synthesis,
ICU syntax validation (Issue 22).

## Design References

- DESIGN §8.1–8.3, §7 (verbatim/limits), §2.2 (plural caveat)
