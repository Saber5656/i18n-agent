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
   - Translatable values must be strings; non-string, non-`@` values (numbers, booleans,
     null — rare but legal JSON) become verbatim entries per DESIGN §7
     (`makeVerbatimEntry` with the raw token text), never `FormatError`.
   - ARB keys are flat identifiers — single `keyPath` segment, no dot splitting.
   - Resource limits via `assertResourceLimits` (5 MiB / 20 000 keys / 10 000-char
     value), one test each.
3. `serialize`:
   - **No-op rule:** when the updated catalog is identical to the parsed one, return
     `existingRaw` unchanged byte-for-byte (satisfies round-trip stability regardless of
     the file's original indentation).
   - Value updates/appends reuse the span-editing approach of the JSON token editor
     (shared `jsonTokenEdit.ts` from Issue 06 — ARB is JSON); indentation is detected,
     not forced.
   - Target files: `@@locale` first (set to target locale), then translated keys in the
     order rule of DESIGN §8.1; the agent **never writes new `@key` metadata into
     targets** (Flutter reads metadata from the template), but pre-existing `@key`
     objects in a target are preserved untouched (not synced, not deleted — fixture
     required). Existing extra `@@` keys in targets preserved.
   - `existingRaw: null` → `{ "@@locale": "<locale>" }` with 2-space indent + trailing
     newline.
4. `defaultPlaceholderProfiles: ["icu", "tags"]`.
5. Fixtures: template with descriptions+placeholders (`{count}` plural example), target
   missing several keys, target with orphan, target containing pre-existing `@key`
   metadata, template with dangling `@meta` (error case), `@@locale` mismatch (error
   case), non-string value file (verbatim case).
6. Conformance harness registered (template and target fixture sets; churn fixtures use
   2-space indent, and the no-op rule covers arbitrary-indent stability).
7. Security note (T-INJ data flow): `@key.description` and placeholder names are
   semi-trusted repo data; this adapter stores them as data only
   (`CatalogEntry.description` / `meta.placeholderNames`) — prompt encoding is Issue
   15's JSON-encoded item contract, and no prompt string concatenation happens here.

## Acceptance Criteria

- [ ] `@key.description` lands in `CatalogEntry.description`; placeholder names extracted
      into `meta.placeholderNames`.
- [ ] Target serialization never adds `@key` metadata; pre-existing target `@key` and
      `@@` keys are preserved byte-identically (fixture-proven).
- [ ] Dangling `@meta` and `@@locale` mismatch raise `FormatError` with key/file named.
- [ ] Non-string values become verbatim entries and round-trip byte-identically.
- [ ] All three resource limits enforced with tests.
- [ ] Conformance harness passes; round-trip of a metadata-rich template is byte-stable
      (no-op rule).

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
