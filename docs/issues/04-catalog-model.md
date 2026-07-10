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
   entries: Map<string, CatalogEntry> }`; helper `makeCatalog(fileId, locale,
   entries: CatalogEntry[])` that flattens keys, rejects duplicates
   (`FormatError` naming the key), and preserves insertion order (Map semantics).
2. `keypath.ts`: `flatKey(parts: string[]): string` joins with `.`, escaping `\` → `\\`
   then `.` → `\.` per segment; `splitKey(flat: string): string[]` exact inverse;
   `isValidSegment` rejects empty segments. Array elements use synthetic segment
   `[<index>]` (e.g. `["colors","[2]"]` → `colors.[2]`); document that `[` needs no
   escaping because segments are escaped wholesale.
3. `hash.ts`: `hashValue(s: string): string` = lowercase hex sha256 of
   `s.normalize("NFC")`, truncated to 32 chars (DESIGN §9.1), via `node:crypto`.
4. `fsSafe.ts`: `assertResourceLimits(input: { bytes: number; keys?: number;
   valueLen?: number; where: string })` enforcing DESIGN §7 caps (5 MiB file, 20 000 keys,
   10 000 chars/value) with `FormatError` messages naming the file and limit.
5. Non-string leaf policy encoded as a type: `RawLeaf = string | number | boolean | null`;
   adapters will mark non-strings via `meta: { verbatim: true }` — define the constant
   `VERBATIM_META_KEY = "verbatim"` here so all adapters share it.
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
- [ ] Module has zero runtime dependencies beyond `node:crypto`.

## Validation

- `npx vitest run tests/unit/core/keypath.test.ts tests/unit/core/catalog.test.ts
  tests/unit/core/hash.test.ts` green; property test iterations visible in output.

## Dependencies

01

## Non-goals

Plural-form modeling (v1 treats plural keys as ordinary keys, DESIGN §2.2), ICU parsing
(Issue 22), serialization (adapters own it).

## Design References

- DESIGN §7 (model, limits, verbatim policy), §9.1 (hash spec)
