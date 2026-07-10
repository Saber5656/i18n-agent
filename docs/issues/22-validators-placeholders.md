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

1. `types.ts`: `Validator` and `ValidationIssue` interfaces verbatim from DESIGN §11.1 —
   note the check input carries `fileId`/`key`/`locale` context (validators stamp them
   into issues; the runner does not post-process) and the adapter-neutral
   `placeholderHints?: string[]` field (populated by the runner from
   `entry.meta.placeholderNames`, e.g. ARB metadata — no ARB-specific naming leaks into
   validator types). The `GlossaryTerm` type is imported from Issue 14's provider types.
   Scope rule: the `placeholders` validator processes only the five token profiles and
   **ignores the `tags` profile entirely** — markup parity is Issue 23's `tags`
   validator.
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
   override or adapter defaults, resolved by caller; `tags` filtered out per req. 1),
   extract token multisets from source and translation; inequality → error issue
   listing missing/extra tokens (sorted, deduped in message). `placeholderHints` add
   expected `icu` names even if absent from source text (covers optional placeholders).
   Unbalanced-brace semantics: if the SOURCE extraction hits `#unbalanced#`, parity is
   skipped for that entry and a warn-severity issue `source-unparseable` is emitted;
   if only the TRANSLATION is unbalanced, parity fails as an error (and `icuSyntax`
   fires independently).
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

- [ ] All five profiles + icuSyntax implemented with the exact token grammars above,
      each with a documented grammar comment. Mechanical linearity checks: no regex
      with nested quantifiers (source review note), no `new RegExp(` in the module
      (`grep` test), and the adversarial timing budget below.
- [ ] Full parity matrix tests pass; messages name missing/extra tokens; unbalanced
      source/translation semantics per requirement 3.
- [ ] Adversarial inputs complete in < 100 ms each (asserted with `performance.now`,
      no fake timers).
- [ ] `placeholderHints` extend expected sets (test uses `meta.placeholderNames`-shaped
      input).
- [ ] Issues carry `fileId`/`key`/`locale` from the check input.

## Validation

- `npx vitest run tests/unit/validate/placeholders.test.ts` green; timing assertions on.
- `grep -R "new RegExp" src/validate` returns nothing.

## Dependencies

04, 05 (PlaceholderProfileId union), 14 (`GlossaryTerm` type referenced by the
Validator interface)

## Non-goals

Full ICU MessageFormat parsing/validation (balance + keywords only), tag parity
(Issue 23), auto-repair of broken placeholders, locale-aware plural category checks.

## Design References

- DESIGN §11.1–11.2, §16 T-REDOS, §8 (per-format default profiles)
