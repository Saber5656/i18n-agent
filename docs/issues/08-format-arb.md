# Title

Flutter ARB format adapter

## Summary

Implement the `arb` FormatAdapter: template-file metadata (`@@locale`, `@key` blocks)
feeding entry descriptions/placeholder hints, translation-only target files.

## Context

ARB is JSON with a metadata convention (DESIGN §8.5). The source (template) file carries
`@key` metadata whose `description` and `placeholders` materially improve LLM translation
quality (they flow into `TranslationItem.description` and validator hints).

## Scope

- In: `src/formats/arb.ts`, registry wiring, fixtures + conformance run.
- Out: `flutter gen-l10n` invocation, ICU plural synthesis, JSON adapter changes.

## Detailed Requirements

1. `options` (strict): none in v1 (`{}` only) — reject unknown options.
2. `parse`:
   - Root must be a JSON object. Keys starting `@@` (e.g. `@@locale`, `@@last_modified`)
     → file-level metadata, never entries. `@@locale`, when present, must equal the
     effective locale (else `FormatError`).
   - Keys starting `@` (e.g. `@title`) attach to their base key: `@title.description` →
     `entry.description`; the full metadata object is kept in `entry.meta.arb` for
     round-trip; `@title.placeholders` keys are stored in `entry.meta.placeholderNames`
     (string[]) for Issue 22's `icu` profile hints.
   - `@x` with no base key `x` → `FormatError` (dangling metadata).
   - Values must be strings (ARB spec); non-strings → `FormatError`.
   - ARB keys are flat identifiers — single `keyPath` segment, no dot splitting.
3. `serialize`:
   - Source-role serialization is never needed beyond round-trip (agent only writes
     targets), but round-trip stability must hold for template files (conformance).
   - Target files: `@@locale` first (set to target locale), then translated keys in the
     order rule of DESIGN §8.1; **no `@key` metadata duplicated into targets** (Flutter
     reads metadata from the template). Existing extra `@@` keys in targets preserved.
   - 2-space indent, trailing newline; `existingRaw: null` → `{ "@@locale": "<locale>" }`.
4. `defaultPlaceholderProfiles: ["icu", "tags"]`.
5. Fixtures: template with descriptions+placeholders (`{count}` plural example), target
   missing several keys, target with orphan, template with dangling `@meta` (error case),
   `@@locale` mismatch (error case).
6. Conformance harness registered (template and target fixture sets).

## Acceptance Criteria

- [ ] `@key.description` lands in `CatalogEntry.description`; placeholder names extracted.
- [ ] Target serialization emits `@@locale` + strings only; template metadata never leaks
      into targets; existing target `@@` keys preserved.
- [ ] Dangling `@meta` and `@@locale` mismatch raise `FormatError` with key/file named.
- [ ] Conformance harness passes; round-trip of a metadata-rich template is byte-stable.

## Validation

- `npx vitest run tests/unit/formats/arb.test.ts tests/formats` green.
- Manual: run against the stock `app_en.arb` from a `flutter create`-generated l10n
  sample (copied fixture) and confirm parse/serialize stability.

## Dependencies

05

## Non-goals

Generating Dart localizations, plural/select expansion, `@@last_modified` maintenance,
non-string ARB extensions.

## Design References

- DESIGN §8.5, §8.1–8.2, §10.3 (description usage)
