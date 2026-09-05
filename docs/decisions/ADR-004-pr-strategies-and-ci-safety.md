# ADR-004: PR strategies, agent-owned branch namespace, and CI safety rails

- Status: Accepted (2026-07-10)
- Deciders: product owner (Round 2: both W1 sync-PR and W2 PR-append in v1), Fable

## Context

Two automation patterns were approved: (W1) push-to-main → dedicated translation PR, and
(W2) append a translation commit to the developer's own PR. Unattended git automation
creates four hazards: merge conflicts on long-lived bot branches, infinite trigger loops,
secret exposure to forks, and destructive pushes to human branches.

## Decision

1. **W1 regenerates, never accumulates:** the work branch `<branchPrefix><base>` is rebuilt
   from the base branch on every run; updates use `git push --force-with-lease`.
2. **Force push is confined to the agent namespace** by a hard-coded, non-configurable
   guard: any forced update to a ref not matching `git.branchPrefix` is refused before the
   remote is touched. Human branches can never be force-pushed by the tool.
3. **W2 appends normal commits** to the current branch only, and refuses to run on
   detached HEAD or on `branchPrefix` branches (self-recursion guard).
4. **Loop prevention is diff-based, not marker-based:** after a translation commit, the
   next triggered run computes an empty diff and exits before any provider call. Templates
   add `concurrency` groups and a `maxKeysPerRun` breaker caps cost per run.
5. **Fork safety:** W2 template runs only for same-repo PRs
   (`head.repo.full_name == github.repository`); `pull_request_target` is forbidden in all
   official docs/templates. Templates request only `contents: write` + `pull-requests: write`.

## Alternatives considered

- **Incremental commits on a persistent bot branch (no force)** — accumulates conflicts as
  base moves; conflict resolution on generated content is wasted work since content is
  machine-regenerable. Rejected.
- **Delete+recreate branch** — closes the open PR and loses review comments. Rejected.
- **Marker/actor-based loop guards as primary defense** — brittle across trigger types;
  kept only as a cheap secondary (commit trailer aids humans and workflow filters).
- **`pull_request_target` to support forks with secrets** — a well-known RCE/secret-exfil
  pattern (untrusted code + secrets). Forks are simply out of scope for W2.

## Consequences

- Bot PRs are always conflict-free and reviewable; review comments survive updates
  (same PR, force-with-lease branch update).
- The "force push" capability exists in code but is provably scoped; Issue 26/36 must
  test the guard directly.
- Fork contributors don't get auto-translations on their PRs (documented; maintainers run
  W1 after merge instead) — accepted product limitation.
