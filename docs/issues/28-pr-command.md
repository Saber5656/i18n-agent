# Title

`pr` command (branch and commit strategies)

## Summary

Implement `i18n-agent pr`: the W1 dedicated-branch strategy (regenerate → force-with-lease
→ create/update PR) and the W2 append strategy (commit to current branch), composing the
pipeline (25), local git (26), and the GitHub adapter (27).

## Context

This command fulfills the README promise ("翻訳してPRを出す"). Its state machine and
guards are specified in DESIGN §12.2 and ADR-004; several are security-contractual
(self-recursion refusal, namespace-confined force push, empty-diff no-op).

## Scope

- In: `src/cli/commands/pr.ts`, strategy orchestration, integration tests against local
  bare-repo origin + nock GitHub API.
- Out: workflow YAML (Issue 31), pipeline/git/API internals.

## Detailed Requirements

1. Flags: all `translate` flags (Issue 25) plus `--strategy branch|commit` (default
   `branch`), `--base <branch>` (branch strategy only).
2. Common preconditions: inside a git repo (else `GitError` exit 4); `GITHUB_TOKEN`/
   `GH_TOKEN` present for `branch` strategy (checked BEFORE any translation spend —
   fail fast, exit 4); provider key presence likewise pre-checked via Issue 14's
   resolver.
3. `--strategy branch` implements DESIGN §12.2 steps 1–6 exactly:
   - clean worktree required (untracked allowed); dirty → `GitError` exit 4;
   - base = `--base` ?? config `git.baseBranch` ?? `defaultBranch()`; `fetch origin base`;
   - work branch `<branchPrefix><base>` (slashes in base kept; ref validated by Issue 26);
   - run pipeline against a checkout of base on the work branch (local `switch -C` from
     `origin/<base>`); empty result → report "up to date", restore original checkout,
     exit 0, **no push, no PR call**;
   - stage exactly: changed target files + lockfile (paths from `PipelineResult`);
     commit (Issue 26 format); push with `forceWithLease: true` (guard active);
   - `ensurePr` (Issue 27) with title/body from config + report; labels from config.
   - Always restore the user's original checkout (branch or SHA) — including on error
     (try/finally; test asserts).
   - `splitPerLocale: true` → loop locales; branch `<prefix><base>-<locale>`; one PR
     per locale; failures in one locale don't abort others (aggregate exit code 2).
4. `--strategy commit` implements DESIGN §12.2:
   - refuse detached HEAD; refuse current branch starting with `branchPrefix`
     (self-recursion guard, T-LOOP) — both `GitError` exit 4 with explanatory message;
   - run pipeline in place; empty → exit 0 silently (loop terminator);
   - stage changed files + lockfile, commit, `push origin HEAD` (NO force of any kind);
   - never calls the PR API.
5. Report: `RunReport.pr` populated on branch strategy (url/number/created);
   `command: "pr"`; JSON/console via shared renderers.
6. Integration tests (local bare origin + nock): branch happy path (assert: bare repo
   has work branch, PR create payload correct, original checkout restored); second run
   with no changes → no push/PR calls (nock strict); second run with new key → lease
   force push + PR update (not create); dirty tree → exit 4 before provider spend
   (provider spy); token missing → exit 4 before provider spend; commit strategy happy
   path appends exactly one commit to current branch; recursion guard (checkout
   `i18n-agent/main` → exit 4); detached HEAD → exit 4; splitPerLocale creates two
   branches/PRs (nock).

## Acceptance Criteria

- [ ] Both strategies match DESIGN §12.2 step-for-step, verified by the integration
      suite above.
- [ ] No provider spend when preconditions fail (spy-proven for token/dirty cases).
- [ ] Empty-diff runs perform zero pushes and zero GitHub API calls.
- [ ] Original checkout always restored (success and thrown-error paths).
- [ ] Commit strategy can never force-push and never touches the PR API (nock strict
      mode).

## Validation

- `npx vitest run tests/integration/pr.test.ts` green, offline (real git + nock).

## Dependencies

25, 26, 27

## Non-goals

Auto-merge, PR comments/reviews, closing stale PRs (logged only), GitLab, scheduling.

## Design References

- DESIGN §12.2, §13, §16 T-LOOP/T-FORCE/T-PRIV
- ADR-004
