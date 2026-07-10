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
   Issue 02): YAML parsed with the same hardened options as Issue 07 (alias cap, no
   custom tags, `.strict()` zod schema):
   ```yaml
   version: 1
   terms:
     - source: "workspace"                  # 1..100 chars, non-empty trimmed
       translations: { ja: "ワークスペース" }  # locale keys validated against config? NO —
                                            # validated as locale-token format only (file
                                            # may serve locales not yet configured)
       note: "optional, ≤200 chars"
   ```
   Caps: ≤ 500 terms; duplicate `source` (case-insensitive) → `ConfigError` naming both
   entries; file ≤ 256 KiB.
2. `loadGlossary(path): Promise<GlossaryTerm[]>` returning Issue 14's `GlossaryTerm`;
   absent config path → `[]` (feature off).
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

- [ ] Glossary schema enforced with all caps; duplicates rejected with both line refs.
- [ ] Relevance filter matches the boundary/substring rules exactly (matrix-tested).
- [ ] No user-controlled regex anywhere in this module (grep + review).
- [ ] Absent glossary/style paths disable features without error.
- [ ] Oversized style guide fails loudly with actionable message.

## Validation

- `npx vitest run tests/unit/glossary` green.

## Dependencies

02, 14 (GlossaryTerm type). Parallel-safe with 15/16.

## Non-goals

CSV/TBX formats, per-file glossaries, term extraction/suggestion, compliance checking
(Issue 23), translation of the note field.

## Design References

- DESIGN §10.5, §16 T-INJ (data flow into prompts), §6.2 (config fields)
