# Title

YAML (Rails / Vue I18n) format adapter

## Summary

Implement the `yaml` FormatAdapter using the `yaml` package Document/CST API so comments
and anchors on untouched nodes survive, with Rails root-locale-key handling and YAML-bomb
hardening.

## Context

Rails `config/locales/*.yml` nests everything under a top-level locale key; Vue I18n YAML
does not. Comment preservation is the differentiator vs naive load/dump (DESIGN §8.4).
YAML parsing of contributor-editable files is a modeled threat (DESIGN §16 T-YAML).

## Scope

- In: `src/formats/yaml.ts`, registry wiring, fixtures + conformance run, security
  fixtures.
- Out: multi-document YAML, custom tags support, Rails pluralization semantics.

## Detailed Requirements

1. Add runtime dep `yaml` (allowlisted, DESIGN §5). Parse with
   `parseDocument(raw, { uniqueKeys: true, maxAliasCount: 100, schema: "core" })`;
   any document error/warning → `FormatError` with line/col. Custom tags (`!ruby/...`
   etc.) → `FormatError` "custom tags are not supported".
2. `options` (strict): `{ rootLocaleKey?: boolean }`, default `true`.
   When `true`: the single top-level map key must equal the file's locale (after
   `localeMap` reversal is handled by the caller passing the effective locale); parse
   strips it from `keyPath`s, serialize re-adds it. Mismatch → `FormatError` naming both.
   When `false`: plain nested map (Vue style).
3. Leaf handling (self-contained; DESIGN §7): string scalars → entries; sequences of
   strings → one entry per element with `[<index>]` segment; non-string scalars
   (number/boolean/null) → `makeVerbatimEntry(keyPath, rawText)` where `rawText` is the
   scalar's source slice from the CST node range (lexical fidelity, e.g. `1.0`, `yes`);
   nested maps recurse into `keyPath` segments. YAML multiline scalars (`|`, `>`) parse
   to their string value; serialization of *updated* multiline values may re-emit as
   plain/quoted scalar (documented acceptable loss — code comment referencing DESIGN §19
   U2). Merge keys (`<<`) are allowed but their alias expansion counts toward
   `maxAliasCount`; exceeding the cap → `FormatError` (bomb defense).
4. `serialize`: mutate the parsed Document in place (`setIn`/`deleteIn`/`addIn`) —
   untouched nodes keep comments/anchors/format; new keys appended at parent-end in
   source order; `prune` deletes orphans. Emit with `doc.toString({ indent: detected })`;
   indent detection: first indented line's leading spaces (2 or 4; fallback 2 — tabs are
   invalid YAML indent and yield `FormatError` from the parser); single trailing newline.
5. `existingRaw: null` → new document with root locale key (if enabled) and detected
   default indent 2.
6. `defaultPlaceholderProfiles: ["rails-percent", "icu", "tags"]`.
7. Security fixtures: alias-expansion bomb (>100 aliases) rejected fast (<1 s, test with
   timer); **merge-key bomb** (`<<` chains multiplying expansion) rejected via the same
   cap; anchor cycle rejected by library — assert `FormatError`, not hang; 5 MiB cap
   enforced before parse.
8. Fixtures: Rails file with comments between keys (`commented-rails.yml`),
   anchors/aliases legitimate use (≤100), one legitimate `<<` merge-key file,
   Vue-style file (`rootLocaleKey: false`), multiline scalars, non-string scalars
   (`1.0`, `true`), unicode. Conformance harness registered via `ConformanceFixture[]`;
   per-fixture `churnBudgetLines` declares tolerated `yaml` emitter normalizations.

## Acceptance Criteria

- [ ] `commented-rails.yml`: after editing exactly the value of `editKey`, every line of
      the file except that value's line(s) is byte-identical (asserted by line diff in a
      unit test — comments, anchors, spacing all intact).
- [ ] Rails root-locale-key stripping/re-adding round-trips; locale mismatch errors.
- [ ] Alias bomb, merge-key bomb, and custom-tag fixtures throw `FormatError` within 1 s.
- [ ] Non-string scalars round-trip byte-identically via raw CST slices
      (`meta.verbatim`).
- [ ] Conformance harness passes with declared churn budgets.
- [ ] `rootLocaleKey: false` mode round-trips a Vue fixture.

## Validation

- `npx vitest run tests/unit/formats/yaml.test.ts tests/formats` green.
- Manual: one-value edit on a commented Rails fixture → `git diff` shows one hunk,
  comments intact.

## Dependencies

05

## Non-goals

Multi-document streams, custom tag support, YAML 1.1 quirks beyond `core` schema, Rails
plural key semantics (keys pass through verbatim), flow-style reformatting guarantees.

## Design References

- DESIGN §8.4, §16 T-YAML, §19 U2
