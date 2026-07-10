# Title

Glossary and style-guide loading

## Summary

Implement loading/validation of the glossary YAML and style-guide markdown referenced by
config, plus the per-batch relevance filter that selects which glossary terms enter a
prompt.

## Context

Glossary and tone steering were selected as v1 quality features (Round 2). The files live
in the user repo (semi-trusted, size-capped) and their content flows into prompts — so
schema validation and caps are security-relevant (DESIGN §10.5, §16 T-INJ).

## Scope

- In: `src/glossary/{types,load}.ts`, relevance filter, style-guide loader, unit tests.
- Out: prompt formatting (Issue 15), glossary compliance validation (Issue 23), CSV
  support (v2).

## Detailed Requirements

1. Glossary file (path from `translation.glossaryPath`, already root-confined by
   Issue 02): YAML parsed with hardened options stated here (self-contained; DESIGN §16
   T-YAML): `parseDocument(raw, { uniqueKeys: true, maxAliasCount: 100, schema:
   "core" })` with a `LineCounter` for positions; parse errors/warnings and custom tags
   → `ConfigError` with line/col; file ≤ 256 KiB checked before parse. The resulting
   plain object is validated by a `.strict()` zod schema:
   ```yaml
   version: 1
   terms:
     - source: "workspace"                  # 1..100 chars, non-empty trimmed
       translations: { ja: "ワークスペース" }  # locale keys validated against config? NO —
                                            # validated as locale-token format only (file
                                            # may serve locales not yet configured)
       note: "optional, ≤200 chars"
   ```
   Caps: ≤ 500 terms (`terms: []` is valid = feature configured but empty); translation
   values non-empty trimmed strings ≤ 200 chars; unknown keys rejected at every level;
   duplicate `source` (case-insensitive) → `ConfigError` naming both term indexes and,
   when available from the LineCounter, both line numbers.
2. Public API (paths arrive absolute + root-confined from Issue 02):
   ```ts
   loadGlossary(path: string | null): Promise<GlossaryTerm[]>      // null → []
   loadStyleGuide(path: string | null): Promise<string | undefined> // null → undefined
   ```
   `GlossaryTerm` is Issue 14's type.
3. Relevance filter `relevantTerms(terms, items, targetLocale): GlossaryTerm[]`
   (DESIGN §10.5): term included iff `translations[targetLocale]` exists AND
   `term.source` occurs in at least one item's `text`, matched case-insensitively with
   word boundaries for sources that are ASCII-word-only, plain substring otherwise
   (CJK rule). Order: by first occurrence order of items (deterministic).
   Linear scan; no regex construction from term text (ReDoS hygiene — use
   `String.prototype.toLowerCase` + `indexOf`, with a manual word-boundary check).
4. Style guide loader: read `translation.styleGuidePath` (root-confined), UTF-8,
   ≤ 16 KiB (larger → `ConfigError` telling the user to trim; NOT silent truncation —
   truncation inside prompts is Issue 15's last-resort only), returned verbatim.
5. Tests: schema happy/sad paths (dup detection, caps, bad locale token, custom tag);
   relevance matrix — ASCII word boundary (`"cat"` does not match `"concatenate"`),
   CJK substring (`"予定"` matches `"予定表"`), case-insensitivity, missing target
   locale excluded, determinism under term permutation; absent files behavior; oversized
   style guide error.

## Acceptance Criteria

- [ ] Glossary schema enforced with all caps and value constraints; duplicates rejected
      naming both term indexes (line numbers when available).
- [ ] Relevance filter matches the boundary/substring rules exactly (matrix-tested).
- [ ] Absent glossary/style paths disable features without error (`[]` / `undefined`).
- [ ] Oversized style guide fails loudly with actionable message.
- [ ] Bomb/tag security fixtures rejected fast (< 1 s) with `ConfigError`.

## Validation

- `npx vitest run tests/unit/glossary` green.
- `grep -R -E "new RegExp" src/glossary` returns nothing (matching uses lowercase +
  indexOf + manual boundary checks only; static regex literals are also absent by
  construction).

## Dependencies

02, 14 (GlossaryTerm type). Parallel-safe with 15/16 (Issue 15 needs only the type,
which lives in 14).

## Non-goals

CSV/TBX formats, per-file glossaries, term extraction/suggestion, compliance checking
(Issue 23), translation of the note field.

## Design References

- DESIGN §10.5, §16 T-INJ (data flow into prompts), T-YAML (hardened parse), T-PATH
  (paths pre-confined by Issue 02), §6.2 (config fields)
