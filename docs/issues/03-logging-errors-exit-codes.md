# Title

Logger, error taxonomy, exit codes, and secret redaction

## Summary

Implement the shared error classes with the exit-code contract of DESIGN §13, a leveled
logger, and a global secret redactor that scrubs credential values from every log and
error line.

## Context

All modules throw these errors; the CLI maps them to exit codes exactly once. Redaction
is a security control (DESIGN §16 T-SECRET): provider keys and tokens must never reach
stdout/stderr, reports, or PR bodies even inside third-party error messages.

## Scope

- In: `src/util/errors.ts`, `src/util/logger.ts`, `src/util/redact.ts`,
  `src/util/env.ts`, top-level CLI error handler wiring, unit tests.
- Out: command implementations, report renderers (Issue 13).

## Detailed Requirements

1. `errors.ts`: base `AppError extends Error { code: ExitCode; kind: string }` and
   subclasses with fixed codes — `ConfigError`(3), `FormatError`(3), `UsageError`(3),
   `EnvError`(4), `GitError`(4), `ProviderError`(5), `LockfileError`(3),
   `PartialFailure`(2). `ProviderError` carries `retryable: boolean` (constructor
   argument, default `false`) — consumed by the batch runner (Issue 16).
   Export `EXIT_CODES` table matching DESIGN §13 (0,1,2,3,4,5,10) with doc comments;
   `exitCodeFor(err: unknown): number` returns `err.code` for `AppError`, else 10.
2. `logger.ts`: levels `error|warn|info|debug`; default `info`; `--verbose`→debug,
   `--quiet`→error; writes to stderr (stdout is reserved for reports/JSON); `--no-color`
   and non-TTY disable ANSI; every emitted string passes through the redactor.
3. `redact.ts`: two registration paths, both feeding one value set:
   - `installRedactor()` collects the **values** of all process env vars whose name
     matches `/(_API_KEY|_TOKEN|_SECRET)$/` plus `GITHUB_TOKEN`/`GH_TOKEN`, length ≥ 8;
   - `registerSecretValue(value: string, source: string)` adds an arbitrary value at
     runtime — the provider credential resolver (Issue 14) MUST call this with the value
     of the configured `apiKeyEnv` (covers arbitrary env var names, DESIGN §10.2).
   `redact(s: string): string` replaces each registered value occurrence with `***`.
   Applied in logger and in the CLI top-level error printer (covers stack traces and
   third-party SDK error messages).
4. `env.ts`: `requireEnv(name: string): string` → value or `EnvError` naming the exact
   variable; `isCI(): boolean` (`CI` env truthy).
5. CLI top-level handler (in `src/cli/index.ts`): `installRedactor()` runs **first**, at
   process start before flag/config parsing, so even early failures are redacted. The
   handler catches anything from commands, prints `error: <redacted message>` (stack only
   with `--verbose`), calls `process.exit(exitCodeFor(err))`. Unexpected errors print a
   bug-report hint and exit 10. Test hook: env `I18N_AGENT_SELFTEST_THROW` (values
   `app:<kind>` or `plain`) makes the CLI throw right after bootstrap — used by exit-code
   and redaction tests against the built binary.
6. Unit tests: exit-code mapping for every class; redactor replaces a fake
   `FAKE_API_KEY=sk-abcdefgh12345678` value embedded in a thrown Error message and in a
   nested stack string; short values (<8 chars) are not redacted (avoid over-matching);
   logger level gating; stdout/stderr separation.

## Acceptance Criteria

- [ ] `EXIT_CODES` contains exactly `{0,1,2,3,4,5,10}` with the DESIGN §13 meanings, and
      each error subclass maps to its specified code (table-driven test).
- [ ] Built CLI with `I18N_AGENT_SELFTEST_THROW=app:provider` exits 5;
      `=plain` exits 10 (execa test against `dist/cli.js`).
- [ ] A provider error whose message embeds the API key prints with `***` in place of the
      key (test proves it), including via `registerSecretValue` for a custom env name.
- [ ] All log output goes to stderr; nothing but reports/JSON goes to stdout.
- [ ] `exitCodeFor` returns 10 for non-AppError values (including strings/undefined).

## Validation

- `npx vitest run tests/unit/util tests/integration/cli-errors.test.ts` green (the
  integration test builds first and drives `dist/cli.js` with
  `I18N_AGENT_SELFTEST_THROW` + `FAKE_API_KEY=sk-test1234567890`, asserting `***` on
  stderr and the exit codes).

## Dependencies

01

## Non-goals

Structured JSON logging, log files, i18n of error messages, telemetry of any kind.

## Design References

- DESIGN §13 (exit codes), §15.1 (quiet/console split), §16 T-SECRET
