# Title

Composite GitHub Action

## Summary

Author the root `action.yml` composite action wrapping the published CLI with pinned
versions, minimal inputs, and correct env wiring — plus its metadata/branding and
usage docs stub.

## Context

The Action is a thin wrapper by decision (ADR-001): environment setup + argument
pass-through only. Version pinning is a supply-chain control (DESIGN §14.1, §16
T-SUPPLY).

## Scope

- In: `/action.yml`, action README section content (full docs in Issue 33), actionlint
  wiring into CI, a self-test workflow job.
- Out: workflow templates for users (Issue 31), release-time version bumping automation
  (Issue 34 — this issue defines the placeholder contract).

## Detailed Requirements

1. `action.yml` metadata: name `i18n-agent`, description (EN), branding
   (icon `globe`, color `blue`). Inputs:
   - `command` (default `pr`; allowed `check|translate|validate|pr` — validated in a
     step, not by trusting the runner),
   - `strategy` (default `branch`; allowed `branch|commit`),
   - `config` (default `i18n-agent.config.json`),
   - `working-directory` (default `.`),
   - `github-token` (default `${{ github.token }}`),
   - `extra-args` (default empty; whitespace-split arguments — see step 3 for the exact
     safe handling; documented as advanced),
   - `package-spec` (default empty = use the baked pin `i18n-agent@EXACT_VERSION`;
     any non-empty value is passed to npx verbatim — a canary version spec like
     `i18n-agent@0.2.0-rc.1` or a local tarball path; DESIGN §14.1).
2. Steps (composite, `shell: bash` each):
   1. `actions/setup-node@<full-SHA>` with `node-version: 22` (comment carries the tag);
   2. input validation step: reject unknown `command`/`strategy` values with a clear
      error (defense against workflow typos silently running `pr`);
   3. run step — exact safe pattern (no `eval`, inputs never interpolated into shell
      source; they arrive as env vars):
      ```yaml
      - shell: bash
        working-directory: ${{ inputs.working-directory }}
        env:
          GITHUB_TOKEN: ${{ inputs.github-token }}
          FORCE_COLOR: "0"
          IN_SPEC: ${{ inputs.package-spec }}
          IN_CMD: ${{ inputs.command }}
          IN_STRATEGY: ${{ inputs.strategy }}
          IN_CONFIG: ${{ inputs.config }}
          IN_EXTRA: ${{ inputs.extra-args }}
        run: |
          spec="${IN_SPEC:-i18n-agent@EXACT_VERSION}"
          read -r -a extra <<< "${IN_EXTRA}"   # whitespace split only; no quoting/
                                               # metachar interpretation (documented)
          args=("$IN_CMD" --config "$IN_CONFIG")
          [ "$IN_CMD" = "pr" ] && args+=(--strategy "$IN_STRATEGY")
          npx --yes "$spec" "${args[@]}" "${extra[@]}"
      ```
      `EXACT_VERSION` is the literal placeholder `0.0.0-managed-by-release` at this
      issue's merge time; Issue 34 owns rewriting it.
3. Provider API keys are NOT inputs — a comment block at the top of `action.yml` states
   they must come from workflow `env:` (DESIGN §14.1 rationale: secret masking + no
   input echoing).
4. CI additions: extend the existing `actionlint` job (Issue 01) to also lint
   `action.yml` + `examples/workflows/*.yml`; a `self-test` job that:
   `npm ci && npm run build && npm pack` → runs the action from the local checkout
   (`uses: ./`) with `package-spec: ${{ github.workspace }}/i18n-agent-0.0.0.tgz` and
   `command: check` against a minimal fixture repo committed under
   `tests/fixtures/action-selftest/` using the `fake` provider — passes on PRs without
   any secret and without the package existing on npm (this is why `package-spec`
   exists). A second self-test step runs a bash-level assertion that `GITHUB_TOKEN` is
   non-empty in the action's env (`test -n "$GITHUB_TOKEN"` inside a step using the
   same env wiring — never printing the value).
5. This issue also appends `"action.yml"` to `package.json` `files[]` (deferred from
   Issue 01), and adds a stub section `## GitHub Action` to `README.md` (one paragraph
   + link placeholder; full docs are Issue 33).
6. Document (inline comments) why `npx` (ADR-001), and the cold-start cost tradeoff.

## Acceptance Criteria

- [ ] `actionlint` clean; tarball-driven self-test job green on a PR without secrets
      and without any npm publish (fake provider).
- [ ] Unknown command/strategy inputs fail with actionable message (self-test negative
      case via a matrix entry expected-to-fail, or unit-style bash test).
- [ ] `GITHUB_TOKEN` env wiring asserted by the bash-level presence check (value never
      printed).
- [ ] No provider key inputs exist; comment block present.
- [ ] `extra-args` handling matches the exact pattern in step 3 (no `eval`, env-var
      passing, whitespace-split documented).
- [ ] All `uses:` pinned by full SHA; `package.json` `files[]` includes `action.yml`.

## Validation

- CI: actionlint + self-test jobs green (the tarball path exercises the real npx flow).
- Post-release canary via `package-spec: i18n-agent@<version>` is recorded in Issue
  34's QA checklist.

## Dependencies

01, 28 (CLI behavior it wraps)

## Non-goals

JS/bundled action, Docker action, caching node_modules, workflow templates (31),
release automation (34).

## Design References

- DESIGN §14.1, §16 T-SUPPLY/T-SECRET
- ADR-001
