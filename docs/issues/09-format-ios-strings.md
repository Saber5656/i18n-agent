# Title

iOS `.strings` format adapter

## Summary

Implement the `ios-strings` FormatAdapter with a hand-rolled parser for the
`"key" = "value";` grammar, comment-to-description mapping, escape handling, and UTF-8
enforcement.

## Context

`.strings` has no maintained npm parser worth trusting at this security tier, and the
grammar is small (DESIGN §8.6). Comments above entries are translator context and feed
`description` → prompt quality.

## Scope

- In: `src/formats/iosStrings.ts`, registry wiring, grammar + escape tests, fixtures,
  conformance run.
- Out: UTF-16 support (v2, DESIGN §19 U4), `.stringsdict` plural files.

## Detailed Requirements

1. Grammar (parser must implement exactly):
   ```
   file     := (comment | entry | blank)*
   comment  := "/*" .*? "*/" | "//" .*? EOL
   entry    := string ws* "=" ws* string ws* ";"
   string   := '"' (escape | [^"\\])* '"'
   escape   := \\ ( " | \\ | n | t | r | U[0-9A-Fa-f]{4,8} )
   ```
   Unknown escape → `FormatError` (line/col). Duplicate key → `FormatError`.
   Keys are single `keyPath` segments (no dot splitting).
   Implementation MUST be a hand-written single-pass scanner (character cursor) — the
   `.*?` in the grammar above is notation, not an implementation license: no
   backtracking regex anywhere in the parser (DESIGN §16 T-REDOS), and adversarial
   fixtures (10 000-char unterminated comment/string) must fail fast (< 100 ms) with
   `FormatError`. Resource limits via `assertResourceLimits` (5 MiB / 20 000 keys /
   10 000-char value), one test each (DESIGN §7).
2. The comment block(s) immediately preceding an entry (no blank line between) become
   `entry.description` (concatenated, `/* */` markers stripped, trimmed).
3. Encoding: reject UTF-16 BOM (`FF FE` / `FE FF`) with actionable message
   `"…is UTF-16; convert to UTF-8 (v1 supports UTF-8 only)"`. Accept UTF-8 with/without
   BOM (preserve BOM presence on serialize).
4. `serialize`: preserve original comments/blank lines/order of untouched entries
   (line-based reconstruction from parse tokens); updated values re-escaped minimally
   (`"`, `\`, newline → `\n`, tab → `\t`); new keys appended at file end in source
   order; orphans kept unless `prune`; final trailing newline.
   New-key comment rule (deterministic): if the source entry has a `description`, emit
   `/* <text> */` on the line above, where `<text>` = description with newlines
   collapsed to single spaces, `*/` replaced by `*∕` (U+2215), truncated to 80 chars
   with a trailing `…` when longer. Entries without a description get **no** comment.
5. `existingRaw: null` → zero-length file content plus entries (no generated header —
   deterministic, snapshot-tested).
6. `defaultPlaceholderProfiles: ["printf-ios"]`.
7. Fixtures: Apple-style file with block comments, `//` comments, escaped quotes/newlines,
   `%@`/`%1$@` placeholders, unicode + emoji, UTF-16 fixture (error case), duplicate-key
   fixture (error case), missing-semicolon fixture (error names line).
8. Conformance harness registered.

## Acceptance Criteria

- [ ] Grammar/escape table implemented exactly; every error case names line/column.
- [ ] Comments above entries become descriptions and survive round-trip byte-identically
      for untouched entries.
- [ ] UTF-16 fixtures rejected with the actionable message; UTF-8 BOM preserved.
- [ ] One-value edit changes only that entry's line(s) (churn test).
- [ ] Adversarial unterminated-comment/string fixtures fail in < 100 ms.
- [ ] New-key comment rule (precedence, truncation, `*/` escaping) snapshot-tested.
- [ ] Conformance harness passes.

## Validation

- `npx vitest run tests/unit/formats/iosStrings.test.ts tests/formats` green, including
  a committed ≥100-entry fixture `tests/fixtures/ios-strings/large-realistic.strings`
  whose unchanged round-trip is asserted byte-identical in a unit test.

## Dependencies

05

## Non-goals

`.stringsdict` plurals, UTF-16, Xcode project integration, placeholder semantics
(Issue 22 owns `printf-ios`).

## Design References

- DESIGN §8.6, §8.1–8.2, §7 (limits), §16 T-REDOS, §19 U4
