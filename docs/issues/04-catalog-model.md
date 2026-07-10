# Title

Catalog data model and key-path utilities

## Summary

Implement the in-memory `Catalog`/`CatalogEntry` model, flat-key escaping utilities with
property-based tests, the NFC hash helper, and shared resource-limit guards used by all
format adapters.

## Context

The catalog is the lingua franca between adapters, diff engine, providers, and validators
(DESIGN §7). Key flattening must be a bijection or lockfile identities silently collide.

## Scope

- In: `src/core/catalog.ts`, `src/core/keypath.ts`, `src/core/hash.ts`,
  `src/util/fsSafe.ts` (size/count guards), unit tests.
- Out: any parsing (Wave 1), diffing (Issue 12).

## Detailed Requirements

1. `catalog.ts`: `CatalogEntry { keyPath: string[]; value: string; description?: string;
   meta?: Record<string, unknown> }`; `Catalog { fileId: string; locale: string;
   entries: Map<string, CatalogEntry>; warnings?: AdapterWarning[] }` with
   `AdapterWarning { code: string; key?: string; message: string }` (DESIGN §7);
   helper `makeCatalog(fileId, locale, entries: CatalogEntry[], warnings?)` that
   flattens keys, rejects duplicates (`FormatError` naming the key), and preserves
   insertion order (Map semantics).
2. `keypath.ts`: `flatKey(parts: string[]): string` joins with `.`, escaping `\` → `\\`
   then `.` → `\.` per segment; `flatKey` throws `FormatError` on empty arrays or empty
   segments. `splitKey(flat: string): string[]` exact inverse; malformed input (empty
   string, trailing lone `\`, empty segment such as `a..b`) → `FormatError`; document
   that `splitKey` is only ever called on `flatKey` output, so these are corruption
   guards, not a parse feature. Array elements use synthetic segment `[<index>]`
   (e.g. `["colors","[2]"]` → `colors.[2]`); `[` needs no escaping because segments are
   escaped wholesale.
3. `hash.ts`: `hashValue(s: string): string` = lowercase hex sha256 of
   `s.normalize("NFC")`, truncated to 32 chars (DESIGN §9.1), via `node:crypto`.
4. `fsSafe.ts`: `assertResourceLimits(input: { bytes: number; keys?: number;
   valueLen?: number; where: string })` enforcing DESIGN §7 caps (5 MiB file, 20 000 keys,
   10 000 chars/value) with `FormatError` messages naming the file and limit.
5. Non-string leaf policy (DESIGN §7): for a non-string leaf, `entry.value` holds the
   **raw lexical source text** of the token (`"1.0"`, `"true"`, `"null"` — never a
   re-serialization) and `entry.meta = { verbatim: true }`. Define
   `VERBATIM_META_KEY = "verbatim"` plus helpers `isVerbatim(entry)` and
   `makeVerbatimEntry(keyPath, rawText)` here so all adapters and the diff engine share
   one representation. Hashing/diffing operate on that raw text.
6. Tests: property-based round-trip `splitKey(flatKey(p)) deep-equals p` (fast-check or
   hand-rolled generator over segments containing `.`, `\`, unicode, emoji; ≥ 1000 cases);
   collision test: `["a.b"]` vs `["a","b"]` produce distinct flat keys; hash: NFC vs NFD
   inputs hash identically, known vector asserted; duplicate-key rejection; resource-limit
   triggers.

## Acceptance Criteria

- [ ] Round-trip property holds for ≥ 1000 generated key paths incl. `.`/`\`/unicode.
- [ ] `flatKey(["a.b"]) !== flatKey(["a","b"])` proven by test.
- [ ] `hashValue("é" NFD) === hashValue("é" NFC)`, 32 lowercase hex chars.
- [ ] Resource limits throw `FormatError` with file name and limit in the message.
- [ ] Malformed `splitKey`/`flatKey` inputs throw `FormatError` (cases from req. 2).
- [ ] No external npm runtime dependencies (Node built-ins and project-local modules
      such as `util/errors.ts` are allowed).

## Validation

- `npx vitest run tests/unit/core/keypath.test.ts tests/unit/core/catalog.test.ts
  tests/unit/core/hash.test.ts tests/unit/util/fsSafe.test.ts` green; property test
  iterations visible in output.

## Dependencies

01, 03 (`FormatError` class)

## Non-goals

Plural-form modeling (v1 treats plural keys as ordinary keys, DESIGN §2.2), ICU parsing
(Issue 22), serialization (adapters own it).

## Design References

- DESIGN §7 (model, limits, verbatim policy), §9.1 (hash spec)
