# Title

Placeholder-parity validators (5 profiles + ICU syntax)

## Summary

Implement the placeholder extraction profiles (`icu`, `i18next`, `rails-percent`,
`printf-ios`, `printf-java`) with multiset-parity checking, and the `icuSyntax` validator
for translated text.

## Context

These validators are the primary machine gate against LLM-corrupted interpolations —
the reason translations can be trusted enough to auto-PR (DESIGN §11.2). Extractors run
on untrusted text, so linear-time behavior is a security requirement (T-REDOS).

## Scope

- In: `src/validate/placeholders.ts` (extractors + `placeholders` validator +
  `icuSyntax` validator), `src/validate/types.ts` (Validator/ValidationIssue interfaces
  per DESIGN §11.1), profile-to-extractor registry, unit tests incl. adversarial cases.
- Out: runner/severity policy (Issue 24), tag parity (Issue 23).

## Detailed Requirements

1. `types.ts`: `Validator` and `ValidationIssue` interfaces verbatim from DESIGN §11.1.
2. Extractors — each a hand-written single-pass scanner (NO backtracking regex; simple
   bounded regexes for token shapes are acceptable if linear), input capped at 10 000
   chars (longer → issue `severity: "error", message: "value exceeds validation cap"`):
   - `icu`: arguments `{name}` and complex arguments `{name, plural|select|selectordinal,
     …}` — extract argument names at nesting depth tracking `{}` balance; unbalanced →
     extraction error token `#unbalanced#`.
   - `i18next`: `{{name}}` (trim inner whitespace; nesting not supported).
   - `rails-percent`: `%{name}` plus Ruby format `%<name>s`.
   - `printf-ios`: `%@ %d %i %u %f %e %g %x %s %ld %lld %.2f`-style incl. positional
     `%1$@` — normalize to `(position?, conversion)` tokens; `%%` ignored.
   - `printf-java`: `%s %d %f %b %n`-style incl. positional `%1$s` and flags/width
     (`%,d`, `%05d`) — normalize like printf-ios; `%%` ignored.
3. `placeholders` validator: for the entry's configured profiles (from file entry
   override or adapter defaults, resolved by caller), extract token multisets from
   source and translation; inequality → error issue listing missing/extra tokens
   (sorted, deduped in message). ARB placeholder hints (`meta.placeholderNames`, Issue
   08) add expected `icu` names even if absent from source text (covers optional
   placeholders).
4. `icuSyntax` validator (translated text only): `{}` balance, argument-name charset
   `[a-zA-Z0-9_]`, known complex keywords (`plural`, `select`, `selectordinal`), plural
   branch braces balanced. Runs only when profiles include `icu`.
5. Adversarial tests (time-bounded): 10 000-char strings of `{{{{…`, `%%%%…`, `{a,{b,{c…`
   nested 1 000 deep — each must complete < 100 ms (assert with performance.now budget)
   and return a deterministic result.
6. Parity test matrix per profile: identical sets pass; missing/extra/renamed tokens
   fail with exact message; reordering passes (multiset semantics); positional printf
   `%1$s %2$s` → `%2$s %1$s` passes; count changes fail; `%%`/escaped forms ignored.
7. Unicode: extraction operates on code points (no surrogate splitting; test emoji
   around placeholders).

## Acceptance Criteria

- [ ] All five profiles + icuSyntax implemented as single-pass scanners with the exact
      token grammars above, each with a documented grammar comment.
- [ ] Full parity matrix tests pass; messages name missing/extra tokens.
- [ ] Adversarial inputs complete within the time budget (CI-asserted).
- [ ] ARB placeholder hints extend expected sets (test with Issue 08 fixture shape).
- [ ] No `RegExp` constructed from input text anywhere in the module.

## Validation

- `npx vitest run tests/unit/validate/placeholders.test.ts` green; timing assertions on.

## Dependencies

04, 05 (PlaceholderProfileId union)

## Non-goals

Full ICU MessageFormat parsing/validation (balance + keywords only), tag parity
(Issue 23), auto-repair of broken placeholders, locale-aware plural category checks.

## Design References

- DESIGN §11.1–11.2, §16 T-REDOS, §8 (per-format default profiles)
