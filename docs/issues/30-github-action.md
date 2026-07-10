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
   - `strategy` (default `branch`),
   - `config` (default `i18n-agent.config.json`),
   - `working-directory` (default `.`),
   - `github-token` (default `${{ github.token }}`),
   - `extra-args` (default empty; appended verbatim — documented as advanced/dangerous
     with a security note),
   - `version` (default empty = use the baked `EXACT_VERSION`; override for canary).
2. Steps (composite, `shell: bash` each):
   1. `actions/setup-node@<full-SHA>` with `node-version: 22` (comment carries the tag);
   2. input validation step: reject unknown `command`/`strategy` values with a clear
      error (defense against workflow typos silently running `pr`);
   3. run step:
      `npx --yes i18n-agent@${{ inputs.version || 'EXACT_VERSION' }} ${{ inputs.command }} …`
      with `--strategy` only when command is `pr`; `--config` always; executed in
      `working-directory`; env: `GITHUB_TOKEN: ${{ inputs.github-token }}`,
      `FORCE_COLOR: 0`.
      `EXACT_VERSION` is a literal placeholder string `0.0.0-managed-by-release` at this
      issue's merge time; Issue 34 owns rewriting it. The run step MUST quote/array-safe
      `extra-args` via `bash -c 'set -- …'` pattern documented in-file.
3. Provider API keys are NOT inputs — a comment block at the top of `action.yml` states
   they must come from workflow `env:` (DESIGN §14.1 rationale: secret masking + no
   input echoing).
4. CI additions: `actionlint` job (pinned) linting `action.yml` +
   `examples/workflows/*.yml`; a `self-test` job that runs the action from the local
   checkout (`uses: ./`) with `command: check` against a minimal fixture repo committed
   under `tests/fixtures/action-selftest/` using the `fake` provider — must pass on PRs
   without any secret.
5. Document (inline comments) why `npx` (ADR-001), and the cold-start cost tradeoff.

## Acceptance Criteria

- [ ] `actionlint` clean; self-test job green on a PR without secrets (fake provider).
- [ ] Unknown command/strategy inputs fail with actionable message (self-test negative
      case via a matrix entry expected-to-fail, or unit-style bash test).
- [ ] `github-token` default works: self-test asserts env visible to CLI (check-only).
- [ ] No provider key inputs exist; comment block present.
- [ ] All `uses:` pinned by full SHA.

## Validation

- CI: actionlint + self-test jobs green.
- Manual: temporarily point `version` input at a published canary (once Issue 34 ships)
  and run `check` in a scratch repo (recorded in Issue 34's QA checklist instead if no
  canary exists yet).

## Dependencies

01, 28 (CLI behavior it wraps)

## Non-goals

JS/bundled action, Docker action, caching node_modules, workflow templates (31),
release automation (34).

## Design References

- DESIGN §14.1, §16 T-SUPPLY/T-SECRET
- ADR-001
