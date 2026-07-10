# Title

Local git operations with safety rails

## Summary

Implement `git/local.ts`: repo introspection, branch management, path-scoped staging,
trailer-stamped commits, and pushes — all via argv-array `execa` with the hard-coded
force-push namespace guard.

## Context

Everything `pr` does to a repo flows through this module; its safety rails (path-scoped
staging, prefix-confined force push, self-recursion refusal) are contractual security
controls (DESIGN §12.1, §16 T-CMD/T-FORCE; ADR-004).

## Scope

- In: `src/git/local.ts`, `src/git/remote.ts` (origin URL → owner/repo parsing), unit +
  integration tests against real temp repos.
- Out: PR API (Issue 27), strategy orchestration (Issue 28).

## Detailed Requirements

1. All invocations: `execa("git", [...args], { cwd, env: minimalEnv })` — never
   `shell: true`, never string interpolation of args (T-CMD). Non-zero exit →
   `GitError` including the git stderr (redacted).
2. API (all async):
   ```ts
   ensureRepo(cwd): Promise<{ root: string }>                    // rev-parse --show-toplevel
   currentBranch(root): Promise<string | null>                   // null = detached
   defaultBranch(root): Promise<string>                          // origin/HEAD, fallback "main" with warn
   isClean(root, opts:{ignoreUntracked:true}): Promise<boolean>  // porcelain v2
   fetch(root, ref): Promise<void>
   createBranchFrom(root, branch, startPoint): Promise<void>     // git switch -C … (local only)
   checkout(root, ref): Promise<void>
   stagePaths(root, paths: string[]): Promise<void>              // git add -- <paths>; rejects [] and paths outside root
   commit(root, message): Promise<{ sha: string }>               // author "i18n-agent <i18n-agent@users.noreply.github.com>" via -c flags
   push(root, branch, opts:{ forceWithLease: boolean; branchPrefix: string }): Promise<void>
   headCommitTouchesOnly(root, paths): Promise<boolean>          // for tests/diagnostics
   ```
3. Commit message format exactly per DESIGN §12.1: subject from config, blank line, body
   line `i18n-agent: auto-translation run`, trailer `X-i18n-agent: <package version>`.
4. **Force guard (T-FORCE)**: `push` with `forceWithLease: true` throws `GitError`
   BEFORE any network call unless `branch.startsWith(branchPrefix)`. The guard reads the
   prefix from an explicit argument (no ambient config) and is not bypassable by any
   flag. Uses `--force-with-lease` only, never `--force`.
5. Ref-name validation: branch names produced elsewhere are validated here with
   `git check-ref-format --branch` rules (reject control chars, `..`, leading `-`, etc.)
   → `GitError` (defense in depth against config-driven ref injection).
6. `stagePaths`: resolves each path, asserts inside root (reuse Issue 02 confinement
   helper), passes after `--` separator. Empty list → `GitError` (programmer error).
7. Identity: commits use `-c user.name=i18n-agent -c user.email=…` so runs don't depend
   on runner git config; committer untouched.
8. Integration tests (real `git` binary in temp dirs — init repo, add origin as local
   bare repo): every API happy path; dirty-tree detection with/without untracked; force
   guard — pushing `i18n-agent/main` with lease succeeds, pushing `main` with
   `forceWithLease: true` throws before remote mutation (assert bare repo ref unchanged);
   ref-name injection attempts (`--upload-pack=…` style names, `-x`, `a..b`) rejected;
   staging path outside root rejected; trailer present in `git log -1 --format=%B`.

## Acceptance Criteria

- [ ] No shell usage anywhere (grep for `shell:` in src/git — none).
- [ ] Force-with-lease outside the prefix is impossible (test proves remote untouched).
- [ ] Malicious ref/path inputs rejected before any git mutation.
- [ ] Commit trailer + author identity exact (byte-compared in test).
- [ ] All functions work on a repo whose default branch is `master` (fallback test).

## Validation

- `npx vitest run tests/integration/git-local.test.ts` green (uses real git; CI provides
  it).

## Dependencies

02, 03

## Non-goals

GitHub API, worktree management, merge/rebase conflict handling (branch strategy
regenerates instead — ADR-004), submodules, signing.

## Design References

- DESIGN §12.1, §16 T-CMD/T-FORCE
- ADR-004
