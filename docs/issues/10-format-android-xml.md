# Title

Android `strings.xml` format adapter

## Summary

Implement the `android-xml` FormatAdapter with hardened XML parsing (no entities/DTD),
`<string>`/`<string-array>` translation, `<plurals>` pass-through with warning,
Android escaping rules, and `values-<locale>` path conventions.

## Context

Android resources are the heaviest v1 format (DESIGN §8.7): XML security hardening is a
modeled threat (T-XXE), Android has its own escape rules, and plurals are explicitly
deferred with a visible warning rather than silent mishandling.

## Scope

- In: `src/formats/androidXml.ts`, registry wiring, security fixtures, conformance run.
- Out: `<plurals>` translation (v2), other resource types (colors, dimens…), resource
  qualifier logic beyond what config `path`/`sourcePath`/`localeMap` already provide.

## Detailed Requirements

1. Add runtime dep `fast-xml-parser` (allowlisted in DESIGN §5; per policy the
   first-using issue edits `package.json` — that is this issue). Parser options
   (security-pinned, test-asserted):
   `processEntities: false`, no DTD processing (reject input containing `<!DOCTYPE` —
   explicit pre-scan → `FormatError "DTD not supported"`), `preserveOrder: true`,
   `commentPropName` enabled, attributes preserved. 5 MiB cap before parse.
2. Root must be `<resources>`. Recognized children:
   - `<string name="k">text</string>` → entry, keyPath `["k"]`.
   - `<string-array name="k"><item>…</item>…</string-array>` → entries `["k","[i]"]`.
   - `<plurals>` → raw block preserved for serialization, excluded from entries, and
     reported via `catalog.warnings` (DESIGN §7 `AdapterWarning`):
     `{ code: "plurals-skipped", key: "<plurals name>", message: … }` — commands map
     these to warn-severity report issues (`validatorId: "format"`).
   - `translatable="false"` on `<string>`/`<string-array>` → excluded from entries,
     preserved on serialize.
   - Comments preceding an element → `entry.description`.
   - Any other element → preserved verbatim, never translated.
3. Text handling — bidirectional conversion table (each row golden-fixture-tested both
   directions):

   | File form (parse →) | Catalog value | (→ serialize) |
   |---|---|---|
   | `&amp; &lt; &gt; &quot; &apos;` | literal char | re-encode `& < >` (always), quotes only where required |
   | `\'` / `\"` / `\n` / `\t` | `'` / `"` / newline / tab | re-escape identically |
   | `\@` / `\?` (first char) | `@` / `?` | re-prefix `\` when first char |
   | `"  padded  "` (quoted form) | text incl. spaces | re-quote when leading/trailing space |
   | `<![CDATA[…]]>` | inner text verbatim | value round-trips inside CDATA |

   Entities beyond the five built-ins are impossible (`processEntities: false` + DTD
   pre-scan rejection).
4. `serialize` preservation rules (per element kind): untouched `<string>`/
   `<string-array>`/`<plurals>`/unknown elements/comments are copied from the original
   raw text via recorded spans (line-based reconstruction — attribute order, whitespace,
   and comments of untouched elements never change); updated `<string>` values replace
   only the element's text content; updated `<string-array>` items replace only that
   `<item>`; new `<string>` elements are appended before `</resources>` in source order
   at 4-space indent; `<?xml version="1.0" encoding="utf-8"?>` header
   preserved/synthesized; `existingRaw: null` → header + `<resources>\n</resources>\n`.
5. `defaultPlaceholderProfiles: ["printf-java", "tags"]`.
6. Path conventions are config's job — document in the issue: source
   `res/values/strings.xml` via `sourcePath`, targets `res/values-{locale}/strings.xml`
   via `path`, `localeMap` e.g. `{"zh-CN": "zh-rCN", "pt-BR": "pt-rBR"}`. Add one
   integration-style test wiring config → adapter paths.
7. Security fixtures: XXE payload (`<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>`)
   → `FormatError`; the test spies on `fs` (`vi.spyOn` of `readFile`/`open`) to assert
   the adapter performs **zero** filesystem reads while parsing in-memory content;
   billion-laughs entity fixture → rejected by DTD pre-scan; oversized file → cap error.
8. Fixtures: plain strings with `%1$s`/`%d`, string-array, plurals block, comments,
   `translatable="false"`, CDATA with `<b>` markup, apostrophes needing escaping.
   Conformance harness registered (churn budgets declared for XML writer normalizations).

## Acceptance Criteria

- [ ] XXE/DTD fixtures are rejected before any entity resolution (fs-spy asserts zero
      filesystem reads during parse).
- [ ] `<plurals>` blocks round-trip byte-identically and appear in `catalog.warnings`
      as `plurals-skipped`.
- [ ] Every row of the escape conversion table has a passing bidirectional
      golden-fixture test (apostrophe fixture round-trips parse→serialize→parse to the
      same value).
- [ ] `translatable="false"` entries never appear in the catalog and survive serialize
      byte-identically.
- [ ] Conformance harness passes; one-value edit on the golden fixture changes ≤ 2
      lines (line-diff assertion).
- [ ] Serialized golden outputs pass `xmllint --noout` (well-formedness only; Android
      resource-compiler validation is a Non-goal).

## Validation

- `npx vitest run tests/unit/formats/androidXml.test.ts tests/formats` green.
- `xmllint --noout` over `tests/fixtures/android/golden/*.xml` in the test run (skip
  with a logged notice if xmllint is absent locally; CI installs it).

## Dependencies

05

## Non-goals

Plurals translation, other Android resource types, density/qualifier resolution,
AAPT validation.

## Design References

- DESIGN §8.7, §16 T-XXE, §2.2 (plurals deferral), §19 U3
