# Title

Tag, empty, and glossary-compliance validators

## Summary

Implement the remaining three validators: `tags` (markup parity incl. react-i18next
numeric tags), `empty`, and `glossary` (heuristic term-compliance warnings).

## Context

Completes the DESIGN §11 validator set. `tags` guards against LLM-mangled inline markup
(also a mild XSS-adjacent control: new tags cannot be silently introduced, DESIGN §16
T-INJ residual); `glossary` closes the loop on Issue 21's steering.

## Scope

- In: `src/validate/{structure.ts,glossaryCheck.ts,registry.ts}` (the registry file
  exports `ALL_VALIDATORS`), unit tests.
- Out: runner (Issue 24), placeholder profiles (Issue 22). Division of labor: Issue 22
  defines profile ids/types and the token-profile validators; this issue implements the
  `tags` validator in `structure.ts` (the DESIGN §11.2 table lists `tags` as a profile
  id — its checking logic lives here by explicit assignment).

## Detailed Requirements

1. `tags` validator (in `structure.ts`): single-pass scanner extracting markup tokens.
   Token grammar: `<` + name + (attribute run: any chars except `<`/`>`) + (`/>` |
   `>`) for opening/self-closing (whitespace before `/>` allowed, so `<br />` is
   self-closing `br`); `</` + name + `>` for closing; numeric react-i18next forms
   `<0>`, `</0>`, `<0/>` (numeric names take no attributes). Name charset
   `[a-zA-Z][a-zA-Z0-9-]*` or `[0-9]+`. Attribute text participates only in span
   scanning, never in parity comparison (`<a href="x">` ≡ `<a href="y">` for parity).
   A `>` inside a quoted attribute value ends the scan at the quote-aware boundary
   (single/double quotes tracked — still single-pass).
   Parity = multiset equality over (kind, name) pairs between source and translation;
   additionally verify open/close balance *within the translation* (well-nesting NOT
   required — UI strings may interleave; balance = equal open/close counts per name).
   Failure lists offending tokens. `<` not forming a token is literal text (no issue).
   10 000-char cap like Issue 22; linear time.
2. `empty` validator: issue when `translated.trim() === ""` while `source.trim() !== ""`;
   also flags translations that are only placeholder tokens when the source contains
   non-placeholder text? — NO (out of scope, keep deterministic; document why in code).
3. `glossary` validator (in `glossaryCheck.ts`): for each `GlossaryTerm` relevant to the
   entry (reuse Issue 21's matcher on the SOURCE text, single entry granularity): if
   `translations[locale]` exists and is not found in the translated text
   (case-insensitive; CJK substring rule), emit issue
   `glossary term "<source>" expected rendering "<target>"`. Severity comes from config
   (`validation.glossary`, default warn) — this validator only reports; severity mapping
   is the runner's job (consistent with DESIGN §11.1).
4. All three implement the `Validator` interface from Issue 22's `types.ts`.
   `src/validate/registry.ts` exports `ALL_VALIDATORS` as a static array in the fixed
   order: placeholders, tags, icuSyntax, empty, glossary (deterministic report order;
   ADR-005 static-registry pattern).
5. Tests: tags matrix (missing close, renamed tag, added tag, numeric tags, self-closing,
   interleaved-but-balanced passes, literal `<` in text, `a < b && b > c` math text no
   false positive); empty matrix (whitespace-only, both-empty passes); glossary matrix
   (present/absent/case/CJK, multiple terms, term relevant but locale missing → no
   issue); adversarial 10 k `<` flood within time budget.

## Acceptance Criteria

- [ ] Tag parity + balance semantics exactly as specified (incl. `<br />`, quoted-`>`
      attributes, attribute-insensitive parity); math/comparison text produces no false
      positives (fixture-proven).
- [ ] Glossary issues carry term and expected rendering; no regex from term text.
- [ ] `src/validate/registry.ts` exports `ALL_VALIDATORS` in the fixed order (test
      asserts ids sequence).
- [ ] 10 000-char `<` flood completes in < 100 ms (`performance.now` assertion, no fake
      timers).

## Validation

- `npm run lint && npm run typecheck && npx vitest run
  tests/unit/validate/structure.test.ts tests/unit/validate/glossaryCheck.test.ts`
  green (the lint/typecheck gate applies to every issue per ISSUE_PLAN §6; restated
  here verbatim).

## Dependencies

21, 22

## Non-goals

HTML sanitization/parsing correctness (token parity only), attribute-value comparison,
well-nesting enforcement, spelling/quality heuristics.

## Design References

- DESIGN §11.1–11.3, §16 T-REDOS
