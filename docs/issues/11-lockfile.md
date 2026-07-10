# Title

Lockfile schema, read/write, and bootstrap semantics

## Summary

Implement `i18n-agent.lock.json` handling: zod schema, deterministic serialization,
corruption handling, bootstrap/adoption updates, and GC of dead entries.

## Context

The lockfile is the durable memory enabling stale detection (DESIGN §9.1, ADR-002). It is
committed to user repos, so serialization must be diff-friendly and reading must treat it
as untrusted data (DESIGN §16 T-LOCK).

## Scope

- In: `src/core/lockfile.ts`, unit tests, corruption fixtures.
- Out: diff classification (Issue 12), CLI flags (`--reset-lockfile` wiring in Issue 25).

## Detailed Requirements

1. Schema (zod, `.strict()` everywhere) exactly per DESIGN §9.1:
   ```ts
   { version: 1,
     files: Record<fileId, { keys: Record<flatKey,
       { sourceHash: string /* ^[0-9a-f]{32}$ */,
         locales: Record<locale, string /* same regex */> }> }> }
   ```
2. API:
   ```ts
   loadLockfile(path: string): Promise<Lockfile>        // absent file → empty {version:1, files:{}}
   saveLockfile(path: string, lf: Lockfile): Promise<void>
   recordTranslation(lf, fileId, flatKey, sourceHash, locale): Lockfile   // pure update
   adoptEntry(lf, fileId, flatKey, sourceHash, locale): Lockfile          // bootstrap rule
   gc(lf, liveKeys: Map<fileId, Set<flatKey>>): Lockfile                  // DESIGN §9.2 GC
   ```
   All update functions are pure (return new object) for testability.
3. `saveLockfile` determinism: keys sorted lexicographically (byte order) at every level
   — files, keys, locales; 2-space indent; trailing newline. Saving twice from permuted
   in-memory maps yields identical bytes (test).
4. Corruption handling: unreadable JSON or schema violation → `LockfileError` (exit 3)
   whose message includes the path, the first schema issue, and the remediation hint
   `run with --reset-lockfile to rebuild`. Never auto-delete a corrupt lockfile.
5. Hash strings are opaque data — never interpolated into shell/paths/prompts (T-LOCK);
   values not matching the regex are schema violations.
6. GC rule (DESIGN §9.2): remove key entries whose flatKey is absent from the live source
   key set **and** absent from every target catalog; remove empty `files` buckets.
7. Size guard: lockfile > 20 MiB → `LockfileError` (defensive cap).
8. Tests: absent-file bootstrap; determinism (permutation test); every corruption fixture
   (bad JSON, wrong version, bad hash format, non-object) → `LockfileError` with hint;
   record/adopt/gc behavior table-tested; GC keeps entries still present in any target.

## Acceptance Criteria

- [ ] Byte-determinism proven by permutation test.
- [ ] Absent lockfile yields empty structure without error (bootstrap path).
- [ ] All corruption fixtures produce `LockfileError` with path + remediation hint.
- [ ] GC removes only entries dead in source AND all targets.
- [ ] No function here performs git or network operations.

## Validation

- `npx vitest run tests/unit/core/lockfile.test.ts` green.
- Manual: hand-edit a fixture lockfile to `"version": 2` → command exits 3 with hint.

## Dependencies

02, 03, 04

## Non-goals

Diff classification, migration of future lockfile versions (v1 knows only version 1),
merge-conflict auto-resolution.

## Design References

- DESIGN §9.1–9.2 (schema, GC), §16 T-LOCK
- ADR-002
