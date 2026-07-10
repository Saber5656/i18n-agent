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
3. Leaf handling identical to Issue 06 (arrays → `[<index>]`, non-strings verbatim via
   the shared meta key). YAML multiline scalars (`|`, `>`) parse to their string value;
   serialization of *updated* multiline values may re-emit as plain/quoted scalar
   (documented acceptable loss — record in code comment referencing DESIGN §19 U2).
4. `serialize`: mutate the parsed Document in place (`setIn`/`deleteIn`/`addIn`) —
   untouched nodes keep comments/anchors/format; new keys appended at parent-end in
   source order; `prune` deletes orphans. Emit with `doc.toString({ indent: detected }
   )`; detect 2/4-space indent like Issue 06; single trailing newline.
5. `existingRaw: null` → new document with root locale key (if enabled) and detected
   default indent 2.
6. `defaultPlaceholderProfiles: ["rails-percent", "icu", "tags"]`.
7. Security fixtures: alias-expansion bomb (>100 aliases) rejected fast (<1 s, test with
   timer); anchor cycle rejected by library — assert `FormatError`, not hang; 5 MiB cap
   enforced before parse.
8. Fixtures: Rails file with comments between keys, anchors/aliases legitimate use
   (≤100), Vue-style file (`rootLocaleKey: false`), multiline scalars, unicode.
   Conformance harness registered; churn budget accounts for the `yaml` emitter's known
   normalizations (declare per-fixture).

## Acceptance Criteria

- [ ] Comments adjacent to untouched keys are byte-preserved after a one-value edit
      (fixture-proven).
- [ ] Rails root-locale-key stripping/re-adding round-trips; locale mismatch errors.
- [ ] Alias bomb and custom-tag fixtures throw `FormatError` within 1 s.
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
