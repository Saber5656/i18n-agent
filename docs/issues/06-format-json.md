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
2. `parse`: implemented on a **span-recording tokenizer** (see req. 4), not bare
   `JSON.parse` — root must be an object (arrays/scalars at root → `FormatError`);
   duplicate keys within an object → `FormatError` naming the key and line. Nested mode:
   recurse objects into `keyPath` segments; arrays of strings → one entry per element
   with `[<index>]` segment (Issue 04); non-string leaves (number/boolean/null) →
   `makeVerbatimEntry(keyPath, rawTokenText)` — `value` is the raw lexical token
   (`1.0` stays `"1.0"`, DESIGN §7). Traversal structures use null-prototype containers
   and own-key iteration only; keys named `__proto__`/`constructor`/`prototype` are
   ordinary data (fixtures required). Flat mode: top-level keys are single segments
   verbatim (no dot splitting; escaping handled by `flatKey`).
3. Indentation detection: inspect the first indented line; support 2/4 spaces or tab;
   default 2 when empty/new. Preserve a single trailing newline always.
4. `serialize` (minimal churn, DESIGN §8.1/§8.3): implement
   `src/formats/jsonTokenEdit.ts`, a small span-based token editor — the tokenizer
   records `{ start, end }` byte spans for every key and value token; value updates
   splice a new string token at the old span; new keys insert before the parent object's
   closing brace (after the last member, adding a comma and the detected indent) in
   source-catalog order, creating intermediate objects as needed; deletions (prune)
   remove the member span incl. its comma. Untouched bytes are copied verbatim —
   escapes, number forms, and spacing of unrelated lines never change. Verbatim entries
   splice their stored raw token text back byte-identically.
5. `existingRaw: null` → `{}`-rooted file built from catalog in source order with detected
   default indent.
6. `defaultPlaceholderProfiles: ["icu", "i18next", "tags"]` (DESIGN §8.3).
7. Resource limits enforced via Issue 04's `assertResourceLimits` — bytes before
   parsing; key count and value length during tokenization; one test per limit
   (5 MiB / 20 000 keys / 10 000-char value).
8. i18next plural suffix keys (`_one`, `_other`, `_zero`, `_two`, `_few`, `_many`) are
   ordinary keys — assert one fixture covers them and document the CLDR caveat in code
   comment referencing DESIGN §2.2.
9. Fixtures (under `tests/fixtures/json/`): nested 2-space, nested 4-space, flat style,
   arrays, non-string leaves incl. `1.0`/`1e3` lexical forms, plural suffixes,
   unicode/emoji values, `__proto__`/`constructor` keys, duplicate-key file (error
   case), a generated ≥200-key realistic catalog `large-realistic.json` (committed) for
   the churn test, and limit-exceeding cases constructed in-test.
10. Register the conformance harness (Issue 05) with `ConformanceFixture[]` entries
    (stable flags, `editKey`, `churnBudgetLines`).

## Acceptance Criteria

- [ ] Conformance harness passes for all fixtures.
- [ ] Editing one value in `large-realistic.json` (≥200 keys) changes ≤ 2 lines
      (asserted by line-diff count in a unit test, not manually).
- [ ] New keys land at parent-end in source order; deep new paths auto-create objects.
- [ ] Flat mode never splits dotted keys; nested mode round-trips `"a.b"` object keys via
      escaping.
- [ ] Verbatim leaves (`1.0`, `true`, `1e3`) are byte-identical after round-trip
      (raw-token splice proven) and carry `meta.verbatim`.
- [ ] Duplicate keys and `__proto__`-style keys behave per requirements 2 (error /
      ordinary data respectively).
- [ ] Indentation (2/4/tab) and single trailing newline preserved per fixture.

## Validation

- `npx vitest run tests/unit/formats/json.test.ts tests/formats` green (includes the
  churn-count assertion on `large-realistic.json`).

## Dependencies

05

## Non-goals

JSON5/JSONC comments (not in v1), key sorting/canonicalization, plural-category synthesis,
ICU syntax validation (Issue 22).

## Design References

- DESIGN §8.1–8.3, §7 (verbatim/limits), §2.2 (plural caveat)
