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

- In: `src/git/local.ts`, unit + integration tests against real temp repos.
- Out: PR API and origin-URL parsing (`src/git/remote.ts` belongs to Issue 27),
  strategy orchestration incl. the commit-strategy self-recursion refusal (Issue 28
  composes it from `currentBranch()` — noted here so no one duplicates it).

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
   addWorktree(root, dir, startPoint): Promise<void>             // git worktree add <dir> <startPoint>
   removeWorktree(root, dir): Promise<void>                      // git worktree remove --force <dir>
   stagePaths(root, paths: string[], allowedPaths: string[]): Promise<void>
     // git add -- <paths>; rejects [], paths outside root, and any path ∉ allowedPaths
     // (callers pass the config-derived target files + lockfile — DESIGN §12.1 rail)
   commit(root, message): Promise<{ sha: string }>
     // author "i18n-agent <i18n-agent@users.noreply.github.com>" via -c flags;
     // `message` is the COMPLETE pre-rendered message (subject, blank line, body,
     // trailer) — rendering happens in Issue 28; this module byte-passes it through
   push(root, branch, opts:{ forceWithLease: boolean; branchPrefix: string }): Promise<void>
   headCommitTouchesOnly(root, paths): Promise<boolean>          // for tests/diagnostics
   ```
   Validation order (uniform for every ref/path-taking function): validate ref names /
   confine paths FIRST, before spawning any git process — `fetch`, `checkout`,
   `createBranchFrom`, `addWorktree`, `push`, and `stagePaths` each have a test proving
   a malicious input is rejected with `GitError` and that no git process was spawned
   (execa spy).
3. Commit message format per DESIGN §12.1 is rendered by callers (Issue 28); this
   module's test byte-compares a full message (subject, exactly one blank line, body
   line, trailer `X-i18n-agent: <version>` in that order) via `git log -1 --format=%B`.
4. **Force guard (T-FORCE)**: `push` with `forceWithLease: true` throws `GitError`
   BEFORE any network call unless `branch.startsWith(branchPrefix)`. The guard reads the
   prefix from an explicit argument (no ambient config) and is not bypassable by any
   flag. Uses `--force-with-lease` only, never `--force`.
5. Ref-name validation: branch names produced elsewhere are validated here with
   `git check-ref-format --branch` rules (reject control chars, `..`, leading `-`, etc.)
   → `GitError` (defense in depth against config-driven ref injection).
6. `stagePaths`: resolves each path, asserts inside root (reuse Issue 02 confinement
   helper) AND membership in `allowedPaths` (exact resolved-path match), passes after
   `--` separator. Empty list → `GitError` (programmer error); a path in the repo but
   outside `allowedPaths` (e.g. `src/index.ts`) → `GitError` (dedicated test).
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
- [ ] Every ref/path-taking function rejects malicious input before spawning git
      (execa-spy tests, one per function).
- [ ] `stagePaths` allowlist rail enforced (in-repo non-allowed file rejected).
- [ ] Commit trailer + author identity exact (byte-compared in test).
- [ ] `defaultBranch`: returns `master` when `origin/HEAD` points at it; returns
      `main` + logs a warning when `origin/HEAD` is unreadable (two separate tests).
- [ ] Worktree add/remove lifecycle works and `removeWorktree` succeeds on a dirty
      temp worktree (`--force`).

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
