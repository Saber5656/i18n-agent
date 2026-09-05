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
   `branch`), `--base <branch>` (branch strategy only; with `--strategy commit` →
   `UsageError` exit 3). Flag semantics on `pr`: `--dry-run` runs the pipeline dry-run
   in place and performs ZERO git mutations and ZERO GitHub API calls (nock-strict
   test); `--prune`, `--reset-lockfile`, `--locale`, `--file`, `--allow-large` pass
   through to the pipeline unchanged.
2. Common preconditions (all checked BEFORE any translation spend; provider spy):
   inside a git repo (else `GitError` exit 4); provider key pre-checked via Issue 14's
   resolver; for `branch` strategy: `GITHUB_TOKEN`/`GH_TOKEN` present (exit 4).
3. `--strategy branch` implements DESIGN §12.2 steps 1–7 (temp-worktree model):
   - base = `--base` ?? config `git.baseBranch` ?? `defaultBranch()`. Validation
     ordering: an explicit `--base`/config value is validated (Issue 26
     `isValidBranchName`) before ANY git command runs; when falling back to
     `defaultBranch()` (itself a read-only git query), the discovered name is validated
     before it is used in `fetch` or anywhere else — option-like values
     (`--upload-pack=…`) rejected with `GitError` in both paths;
   - `fetch origin <base>`; work branch `<branchPrefix><base>`;
   - `addWorktree(tmpdir, origin/<base>)` (Issue 26) — the user's checkout is never
     touched, so NO clean-worktree precondition; run the pipeline with root = tmpdir;
   - empty pipeline result → remove worktree, report "up to date", exit 0, **no push,
     no PR call** (nock strict);
   - stage in the worktree exactly the changed target files + lockfile
     (`stagePaths(paths, allowedPaths)` from `PipelineResult`); commit with the
     rendered DESIGN §12.1 message; push with `forceWithLease: true` (prefix guard
     active);
   - `ensurePr` (Issue 27) with title/body from config + report; labels from config;
   - try/finally: `removeWorktree` always runs (success and error paths; test asserts
     no stale worktrees via `git worktree list`).
   - `splitPerLocale: true` → loop locales; branch `<prefix><base>-<locale>`; one PR
     per locale; one locale's failure doesn't abort others.
4. `--strategy commit` implements DESIGN §12.2:
   - refuse detached HEAD; refuse current branch starting with `branchPrefix`
     (self-recursion guard, T-LOOP); refuse when any config-managed path (resolved
     target files or lockfile) has local modifications (porcelain check limited to
     those paths) — all `GitError` exit 4 with explanatory messages;
   - run pipeline in place; empty → exit 0 silently (loop terminator);
   - stage config-managed changed paths only, commit, `push origin HEAD` (NO force);
   - never calls the PR API.
5. Report: `command: "pr"`; single-PR runs populate `RunReport.pr`; `splitPerLocale`
   runs populate `RunReport.prs[]` with one `PrRef` per **successfully created or
   updated PR** (`pr` omitted — DESIGN §15.2); a locale that produced no PR (pipeline
   or git/API failure) appears instead in `failures[]` as
   `{ id: "<locale>", reason: … }`. Exit-code aggregation across locales:
   highest-severity result wins (10 > 5 > 4 > 3 > 2 > 0).
6. Integration tests (local bare origin + nock): branch happy path (assert: bare repo
   has work branch, PR create payload correct, user checkout byte-untouched, no stale
   worktree); branch strategy with a DIRTY user tree still succeeds (worktree model);
   second run with no changes → no push/PR calls (nock strict); second run with new key
   → lease force push + PR update (not create); malicious `--base` rejected before any
   git spawn; token missing → exit 4 before provider spend (provider spy); `--dry-run`
   → zero git mutations/API calls; commit strategy happy path appends exactly one
   commit to current branch; commit strategy with locally-modified lockfile → exit 4;
   recursion guard (checkout `i18n-agent/main` → exit 4); detached HEAD → exit 4;
   splitPerLocale creates two branches/PRs with `prs[]` populated and aggregated exit
   code per requirement 5 (one locale forced to fail).

## Acceptance Criteria

- [ ] Branch strategy: base validated pre-git; worktree created from `origin/<base>`;
      empty result → zero push/API; staged paths ⊆ allowlist; lease-force push only on
      prefix branch; PR create-vs-update both covered; worktree always removed
      (each item is a named integration test).
- [ ] Commit strategy: detached-HEAD, recursion, and dirty-managed-path refusals; one
      plain commit; plain push; zero PR API calls (nock strict).
- [ ] No provider spend when preconditions fail (spy-proven for token/guard cases).
- [ ] User checkout is never modified by branch strategy (dirty-tree run passes with
      tree hash unchanged).
- [ ] `prs[]`/exit-code aggregation for splitPerLocale per requirement 5.

## Validation

- `npx vitest run tests/integration/pr.test.ts` green, offline (real git + nock).

## Dependencies

25, 26, 27

## Non-goals

Auto-merge, PR comments/reviews, closing stale PRs (logged only), GitLab, scheduling.

## Design References

- DESIGN §12.2, §13, §16 T-LOOP/T-FORCE/T-PRIV
- ADR-004
