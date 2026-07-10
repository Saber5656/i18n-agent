# i18n-agent — v1 Issue Plan

Derived from [docs/DESIGN.md](./DESIGN.md). GitHub Issues are generated 1:1 from
`docs/issues/NN-short-title.md`; this file is the roadmap/grouping layer and must be
updated **before** GitHub Issues when they diverge.

## 1. v1 Completion Statement

v1 is complete when **all issues 01–36 are closed with their Acceptance Criteria and
Validation sections satisfied**. At that point the following is true, with no product
behavior left outside the issue set:

- `npm install -g i18n-agent` (or `npx i18n-agent`) provides `init`, `check`, `translate`,
  `validate`, and `pr` with the exit-code contract of DESIGN §13.
- Missing and stale UI strings in JSON / YAML / ARB / iOS `.strings` / Android
  `strings.xml` catalogs are detected via the committed lockfile (DESIGN §9), translated
  through OpenAI / Anthropic / Gemini / Ollama / fake (DESIGN §10), validated (DESIGN §11),
  and published as a GitHub PR under both W1 (branch) and W2 (append) strategies (DESIGN §12).
- The composite GitHub Action and the two official hardened workflow templates work as
  documented (DESIGN §14), and every threat in DESIGN §16 has a passing verification test
  or documented control (Issue 36 gate).
- README + user docs, SECURITY.md/CONTRIBUTING.md, CI, and the changesets release pipeline
  with npm provenance are in place (DESIGN §17–18).

Remaining exceptions are only the Known Unknowns of §8 below, which may spawn new issues
during implementation.

## 2. Issue List (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-repo-scaffold.md` | Repository scaffold, toolchain, and CI | 0 |
| 02 | `issues/02-config-schema-loader.md` | Config schema, loader, and path confinement | 0 |
| 03 | `issues/03-logging-errors-exit-codes.md` | Logger, error taxonomy, exit codes, secret redaction | 0 |
| 04 | `issues/04-catalog-model.md` | Catalog data model and key-path utilities | 0 |
| 05 | `issues/05-format-adapter-interface.md` | FormatAdapter interface, registry, conformance harness | 1 |
| 06 | `issues/06-format-json.md` | JSON (i18next family) adapter | 1 |
| 07 | `issues/07-format-yaml.md` | YAML (Rails / Vue I18n) adapter | 1 |
| 08 | `issues/08-format-arb.md` | Flutter ARB adapter | 1 |
| 09 | `issues/09-format-ios-strings.md` | iOS `.strings` adapter | 1 |
| 10 | `issues/10-format-android-xml.md` | Android `strings.xml` adapter | 1 |
| 11 | `issues/11-lockfile.md` | Lockfile schema, read/write, bootstrap semantics | 2 |
| 12 | `issues/12-diff-engine.md` | Missing/stale/orphan diff engine | 2 |
| 13 | `issues/13-check-command.md` | `check` command with console/JSON reports | 2 |
| 14 | `issues/14-provider-contract.md` | Provider interface, registry, fake provider | 3 |
| 15 | `issues/15-prompt-builder.md` | Prompt builder with injection defenses | 3 |
| 16 | `issues/16-batch-retry.md` | Batching, retry, and concurrency orchestration | 3 |
| 17 | `issues/17-provider-openai.md` | OpenAI provider | 3 |
| 18 | `issues/18-provider-anthropic.md` | Anthropic provider | 3 |
| 19 | `issues/19-provider-gemini.md` | Google Gemini provider | 3 |
| 20 | `issues/20-provider-ollama.md` | Ollama provider | 3 |
| 21 | `issues/21-glossary-style.md` | Glossary and style-guide loading | 3 |
| 22 | `issues/22-validators-placeholders.md` | Placeholder-parity validators (5 profiles + ICU syntax) | 4 |
| 23 | `issues/23-validators-structure.md` | Tag/empty/glossary-compliance validators | 4 |
| 24 | `issues/24-validation-runner.md` | Validation runner and `validate` command | 4 |
| 25 | `issues/25-translate-pipeline.md` | Translation pipeline and `translate` command | 5 |
| 26 | `issues/26-git-local-ops.md` | Local git operations with safety rails | 6 |
| 27 | `issues/27-github-pr-adapter.md` | GitHub PR adapter (Octokit) | 6 |
| 28 | `issues/28-pr-command.md` | `pr` command (branch/commit strategies) | 6 |
| 29 | `issues/29-init-command.md` | `init` command | 6 |
| 30 | `issues/30-github-action.md` | Composite GitHub Action | 7 |
| 31 | `issues/31-workflow-templates.md` | Official hardened workflow templates | 7 |
| 32 | `issues/32-e2e-tests.md` | End-to-end fixture test suite | 7 |
| 33 | `issues/33-user-docs.md` | README and user documentation | 8 |
| 34 | `issues/34-release-pipeline.md` | Versioning, publishing, provenance, action tagging | 8 |
| 35 | `issues/35-community-security-files.md` | SECURITY.md, CONTRIBUTING.md, repo hygiene | 8 |
| 36 | `issues/36-security-audit-gate.md` | Security acceptance audit vs threat model | 8 |

## 3. Dependency Table

`A ← B` means B depends on A. Transitive deps omitted.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01, 03 (`ConfigError` class) |
| 03 | 01 |
| 04 | 01, 03 (`FormatError` class) |
| 05 | 03, 04 |
| 06–10 | 05 |
| 11 | 02, 03, 04 |
| 12 | 04, 11 |
| 13 | 02, 03, 05, 06 (json fixtures only), 12 |
| 14 | 02, 03 |
| 15 | 14 (GlossaryTerm type is defined in 14; loader lands later in 21) |
| 16 | 03, 12 (DiffItem type), 14, 15 (response parser) |
| 17–20 | 14, 15, 16 |
| 21 | 02, 14 |
| 22 | 04, 05 (profile ids), 14 (GlossaryTerm type) |
| 23 | 21, 22 |
| 24 | 02, 13 (report shapes), 21 (glossary loader), 22, 23 |
| 25 | 12, 13, 14 (provider registry), 16, 21, 24, 06–10 (all adapters for full matrix; runnable with 06 alone) |
| 26 | 02, 03 |
| 27 | 02, 03, 13 (`RunReport` type for PR body) |
| 28 | 25, 26, 27 |
| 29 | 02, 03 |
| 30 | 28 (published CLI behavior), 01 |
| 31 | 30 |
| 32 | 25, 28 (uses fake provider incl. its call-counter hook from 14, bare-repo origin, nock GitHub) |
| 33 | 28, 29, 30, 31 |
| 34 | 01, 30 |
| 35 | 01 |
| 36 | 02, 03, 07, 10, 15, 25, 26, 31, 32, 33, 34, 35 (audits their controls; runs last) |

Parallelism notes: within Wave 1, issues 06–10 are mutually independent; within Wave 3,
17–20 are mutually independent and 21 can run parallel to 15/16; 26/27/29 are mutually
independent in Wave 6.

## 4. Implementation Waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Foundation | 01–04 | CI green; config/catalog/error primitives merged |
| 1 | Format adapters | 05–10 | all 5 adapters pass the shared conformance harness |
| 2 | Detection | 11–13 | `check` correct on fixture repos, exit-code contract met |
| 3 | Translation | 14–21 | fake+4 real providers pass mocked contract tests |
| 4 | Validation | 22–24 | `validate` command; error-severity rejection works |
| 5 | Pipeline | 25 | `translate` end-to-end with fake provider on all formats |
| 6 | Git & PR | 26–29 | `pr` both strategies against local bare repo + mocked API |
| 7 | CI surface | 30–32 | Action + templates + full e2e suite |
| 8 | Ship | 33–36 | docs, release pipeline, security gate all green |

## 5. Coverage Table (DESIGN.md § → issues)

| DESIGN section | Covered by |
|---|---|
| §2 scope/non-goals | plan-level (this file); enforced via issue Non-goals sections |
| §3 workflows W1/W2/W3 | 25, 28, 30, 31, 13, 24, 29 |
| §4 architecture / §5 layout | 01 (skeleton), all module issues |
| §6 configuration | 02, 29 |
| §7 catalog model | 04 |
| §8.1–8.2 adapter contract | 05 |
| §8.3–8.7 formats | 06, 07, 08, 09, 10 |
| §9 lockfile & diff | 11, 12 |
| §10.1–10.2 provider contract/credentials | 14 |
| §10.3 prompt | 15 |
| §10.4 batching/retry | 16 |
| §10.2+§10.4 concrete providers | 17, 18, 19, 20 |
| §10.5 glossary/style | 21 |
| §11 validation | 22, 23, 24 |
| §12.1 local git | 26 |
| §12.2 strategies | 28 |
| §12.3 GitHub adapter | 27 |
| §13 CLI surface | 13, 24, 25, 28, 29 (+03 exit codes) |
| §14 Action & templates | 30, 31 |
| §15 reporting | 13 (console/json), 27+28 (PR body), 03 (redaction) |
| §16 security model | 02, 03, 07, 10, 15, 25, 26, 28, 31, 34, 35; audited end-to-end by 36 |
| §17 testing strategy | distributed per issue Validation sections; integration/e2e in 32 |
| §18 release | 34 |
| §19 known unknowns | §8 below |

Every DESIGN section that describes product behavior maps to at least one issue; sections
§1–§2 are definitional and enforced through issue Scope/Non-goals.

## 6. Whole-Product Validation Strategy

1. **Per-issue gates:** each issue's Validation section names exact commands
   (`npm run lint && npm run typecheck && npx vitest run <paths>`); an issue is not done
   until they pass in CI.
2. **Conformance harness (Issue 05)** keeps the five adapters behaviorally identical on
   the shared contract (round-trip, append order, orphans, caps).
3. **Contract tests with mocked HTTP** for all providers; CI never calls live APIs.
4. **Integration/E2E (Issue 32):** fixture repos × 5 formats, fake provider, local bare
   git origin, nock-mocked GitHub API; asserts files, lockfile, exit codes, PR payloads,
   loop termination (second run = no provider calls).
5. **Security gate (Issue 36):** every DESIGN §16 threat row must point to a passing test
   or a documented, reviewed control before v1 release.
6. **Manual QA checklist (Issue 34):** one live W1 + W2 run on a scratch repo with a real
   provider before first `1.0.0` tag; results recorded in the release PR.

## 7. Deferred to v2 (explicitly out of v1 issues)

Source-code scanning & key extraction; translation memory/cache; gettext PO; GitLab/
Bitbucket adapters; DeepL/machine-translation providers; CLDR plural-category expansion
(i18next suffixes, Android `<plurals>` translation); CSV glossary; UTF-16 `.strings`;
dynamic plugin system (needs its own ADR); interactive `init` wizard; per-locale length/
QA heuristics; GHES verification.

## 8. Known Unknowns (tracked; may create issues mid-implementation)

Mirrors DESIGN §19: U1 provider CJK quality tuning · U2 YAML CST fidelity limits ·
U3 Android XML writer whitespace policy · U4 UTF-16 `.strings` demand · U5 real-GitHub
Action E2E automation · U6 rate-limit behavior on very large backfills (chunked resume) ·
U7 GHES compatibility · U8 npm account provenance/2FA specifics.
Discovery rule: when an unknown materializes, add a `docs/issues/NN-*.md` draft first,
update this plan, then open the GitHub Issue.
