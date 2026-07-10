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

1. Add runtime dep `fast-xml-parser`. Parser options (security-pinned, test-asserted):
   `processEntities: false`, no DTD processing (reject input containing `<!DOCTYPE` —
   explicit pre-scan → `FormatError "DTD not supported"`), `preserveOrder: true`,
   `commentPropName` enabled, attributes preserved. 5 MiB cap before parse.
2. Root must be `<resources>`. Recognized children:
   - `<string name="k">text</string>` → entry, keyPath `["k"]`.
   - `<string-array name="k"><item>…</item>…</string-array>` → entries `["k","[i]"]`.
   - `<plurals>` → kept verbatim in `meta`, excluded from entries, surfaced by parse as
     `pluralsSkipped` (adapter returns them via `Catalog` meta convention
     `catalog-level meta` documented in code; pipeline reports warning per DESIGN §8.7).
   - `translatable="false"` on `<string>`/`<string-array>` → excluded from entries,
     preserved on serialize.
   - Comments preceding an element → `entry.description`.
   - Any other element → preserved verbatim, never translated.
3. Text handling: decode only the five XML built-ins (`&amp; &lt; &gt; &quot; &apos;`)
   manually (entities are off); CDATA sections supported read-only (values containing
   markup round-trip inside CDATA). Android escapes on **serialize**: `'` → `\'`,
   `"` → `\"`, literal newline → `\n`, `@`/`?` first-char → prefix `\`, leading/trailing
   whitespace → wrap value in double quotes (per Android resource rules).
4. `serialize`: line-based minimal churn like other adapters; new `<string>` elements
   appended before `</resources>` in source order, 4-space indent (Android convention),
   `<?xml version="1.0" encoding="utf-8"?>` header preserved/synthesized;
   `existingRaw: null` → header + empty `<resources/>` expanded form.
5. `defaultPlaceholderProfiles: ["printf-java", "tags"]`.
6. Path conventions are config's job — document in the issue: source
   `res/values/strings.xml` via `sourcePath`, targets `res/values-{locale}/strings.xml`
   via `path`, `localeMap` e.g. `{"zh-CN": "zh-rCN", "pt-BR": "pt-rBR"}`. Add one
   integration-style test wiring config → adapter paths.
7. Security fixtures: XXE payload (`<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>`)
   → `FormatError`, file content never read; billion-laughs entity fixture → rejected by
   DTD pre-scan; oversized file → cap error.
8. Fixtures: plain strings with `%1$s`/`%d`, string-array, plurals block, comments,
   `translatable="false"`, CDATA with `<b>` markup, apostrophes needing escaping.
   Conformance harness registered (churn budgets declared for XML writer normalizations).

## Acceptance Criteria

- [ ] XXE/DTD fixtures are rejected before any entity resolution (test asserts message
      and that no fs access occurred — use a nonexistent path in payload).
- [ ] `<plurals>` blocks round-trip byte-identically and are reported as skipped.
- [ ] Escaping rules produce Android-valid output (apostrophe fixture round-trips through
      parse→serialize→parse to the same value).
- [ ] `translatable="false"` entries never appear in the catalog and survive serialize.
- [ ] Conformance harness passes; one-value churn budget met.

## Validation

- `npx vitest run tests/unit/formats/androidXml.test.ts tests/formats` green.
- Manual: adapter output for a modified fixture opens clean in Android Studio's XML lint
  (spot check) or `xmllint --noout`.

## Dependencies

05

## Non-goals

Plurals translation, other Android resource types, density/qualifier resolution,
AAPT validation.

## Design References

- DESIGN §8.7, §16 T-XXE, §2.2 (plurals deferral), §19 U3
