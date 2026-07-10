# ADR-002: Committed lockfile hash ledger for change detection

- Status: Accepted (2026-07-10)
- Deciders: product owner (Round 2: stale re-translation selected for v1), Fable (design)

## Context

Detecting *missing* keys needs only a set difference between source and target catalogs.
Detecting *stale* translations (source text edited after translation) requires knowing
what the source text was when each translation was produced. That state must live
somewhere durable, shared by CI and local runs.

## Decision

Persist a **lockfile `i18n-agent.lock.json`, committed to the user repository**, mapping
`fileId → flatKey → { sourceHash, locales: { <target>: sourceHashAtTranslation } }`
(sha256 over NFC-normalized value, hex truncated to 32 chars). Diff semantics and the
bootstrap rule ("adopt existing translations, never mass-retranslate on first run") are
specified in DESIGN §9.

## Alternatives considered

- **Git history comparison** (diff source file against last agent commit) — breaks on
  squash/rebase/shallow clones, couples core logic to git availability, and cannot express
  per-target-locale freshness.
- **No state / heuristic matching** — cannot distinguish "source edited" from "translation
  intentionally different"; silent staleness was rejected by requirements (Round 2).
- **State outside the repo** (cache dir, hosted store) — CI runners are ephemeral; a hosted
  store contradicts the no-telemetry/no-service posture.

## Consequences

- Stale detection is deterministic, offline, and survives history rewrites.
- The lockfile appears in diffs/PRs; deterministic sorted serialization keeps that noise
  minimal and reviewable.
- Tampered/corrupt lockfiles are a modeled threat (DESIGN §16 T-LOCK): schema-validated,
  worst case is re-translation, never code execution.
- Bootstrap adoption means pre-existing wrong translations are trusted initially; users
  fix by editing source (making entries stale) or `--reset-lockfile`.
