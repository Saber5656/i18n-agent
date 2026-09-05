# Title

Official hardened workflow templates

## Summary

Author the two supported user-facing workflow templates — `sync-pr.yml` (W1) and
`pr-append.yml` (W2) — with every security guard from the design, inline rationale
comments, and lint coverage.

## Context

Templates are where users' CI security posture is actually decided; they encode ADR-004's
guards (same-repo condition, no `pull_request_target`, minimal permissions, concurrency,
loop safety) as copy-pasteable YAML (DESIGN §14.2, §16 T-FORK/T-LOOP/T-PRIV).

## Scope

- In: `examples/workflows/sync-pr.yml`, `examples/workflows/pr-append.yml`, inline
  security comments, actionlint coverage (CI wiring exists from Issue 30).
- Out: action implementation (30), user-docs prose (33 — but templates must be
  self-explanatory standalone).

## Detailed Requirements

1. `sync-pr.yml` (W1), exact spec:
   - `on: push: branches: [main]` with adjacent comment `# ← replace "main" with your
     default branch if it differs`; `paths:` actively lists `i18n-agent.config.json`,
     `i18n-agent.lock.json`, and the **uncommented** example glob `locales/en.json`
     with comment `# ← edit to match your source-locale file(s)` (an inactive filter
     that never triggers is worse than an example users must adjust);
   - top-level `permissions: contents: write, pull-requests: write` and nothing else;
   - `concurrency: { group: i18n-agent-sync, cancel-in-progress: false }` (queued, not
     cancelled — cancelling mid-push risks orphan branches; comment explains);
   - single job: checkout (pinned SHA, `fetch-depth: 0` with comment why — base
     resolution), `uses: Saber5656/i18n-agent@v1` with `command: pr`,
     `strategy: branch`; `env: OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}` plus
     commented alternates for the other three providers.
2. `pr-append.yml` (W2), exact spec:
   - `on: pull_request: types: [opened, synchronize]`;
   - job-level `if:` exactly:
     `github.event.pull_request.head.repo.full_name == github.repository && !startsWith(github.head_ref, 'i18n-agent/')`
     with a comment block explaining fork-secret semantics and the recursion guard;
   - `permissions: contents: write` only (no PR API used by commit strategy);
   - `concurrency: { group: i18n-agent-${{ github.ref }}, cancel-in-progress: true }`;
   - checkout with `ref: ${{ github.event.pull_request.head.ref }}` (pinned SHA);
   - action with `command: pr`, `strategy: commit`; the same provider `env:` block as
     W1 (`OPENAI_API_KEY` from secrets + commented alternates — the commit strategy
     still translates and needs the key).
   Action-reference pinning policy (T-SUPPLY exception, stated in a comment in both
   templates): `Saber5656/i18n-agent@v1` is a mutable major tag **by design** for user
   templates — the executed CLI is contained by the SHA-pinned action's baked
   `EXACT_VERSION`; users wanting the strictest posture may pin the action by full SHA
   (comment shows how). Third-party actions (checkout, setup-node) ARE SHA-pinned in
   the templates.
3. Both templates: header comment box — what the workflow does, required secrets, the
   sentence "Never change `on:` to `pull_request_target`; doing so exposes secrets to
   untrusted code" (T-FORK), and a link placeholder to docs (Issue 33 fills URL).
4. Loop-termination note comment in W2: the synchronize event fired by the agent's own
   commit results in an empty diff and a no-op exit (DESIGN §12.2) — one extra cheap run
   is expected and accepted.
5. Both pass `actionlint` with zero warnings; YAML anchors not used (copy-paste
   friendliness).
6. Negative-example file `examples/workflows/DO-NOT-pull_request_target.md` — a short
   markdown explaining the anti-pattern with a minimal exploit narrative (no working
   exploit code), linked from both templates' comments.

## Acceptance Criteria

- [ ] Mechanical policy test (`tests/security/workflow-templates.test.ts`) asserts on
      both template files: required trigger/permissions/concurrency keys present with
      the exact values above; the W2 `if:` guard string present verbatim;
      `pull_request_target` absent from the entire repo (`git grep`); no YAML anchors;
      third-party `uses:` are 40-char SHA-pinned; required comment phrases present
      (`replace "main"`, `edit to match your source-locale`, fork-secret note,
      pin-by-SHA note).
- [ ] actionlint zero findings in CI for both templates.
- [ ] The anti-pattern doc exists and is referenced by both templates.

## Validation

- `npx actionlint examples/workflows/sync-pr.yml examples/workflows/pr-append.yml`
  clean locally and in CI; `npx vitest run tests/security/workflow-templates.test.ts`
  green.
- Live no-op run of `sync-pr.yml` in a scratch repo is recorded in Issue 34's QA
  checklist (not a completion gate here).

## Dependencies

30

## Non-goals

Cron/schedule template (documented as a variant in Issue 33 prose only), GitLab CI,
reusable-workflow packaging, org-wide rulesets.

## Design References

- DESIGN §14.2, §16 T-FORK/T-LOOP/T-PRIV
- ADR-004
