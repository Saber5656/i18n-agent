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
   `PartialFailure`(2). Export `EXIT_CODES` table matching DESIGN §13 (0,1,2,3,4,5,10)
   with doc comments; `exitCodeFor(err: unknown): number` returns `err.code` for
   `AppError`, else 10.
2. `logger.ts`: levels `error|warn|info|debug`; default `info`; `--verbose`→debug,
   `--quiet`→error; writes to stderr (stdout is reserved for reports/JSON); `--no-color`
   and non-TTY disable ANSI; every emitted string passes through the redactor.
3. `redact.ts`: `installRedactor(patterns: string[])` collects the **values** of all
   process env vars whose name matches `/(_API_KEY|_TOKEN|_SECRET)$/` plus
   `GITHUB_TOKEN`/`GH_TOKEN`, with length ≥ 8; `redact(s: string): string` replaces each
   value occurrence with `***`. Applied in logger and in the CLI top-level error printer
   (covers stack traces and third-party SDK error messages).
4. `env.ts`: `requireEnv(name: string): string` → value or `EnvError` naming the exact
   variable; `isCI(): boolean` (`CI` env truthy).
5. CLI top-level handler (in `src/cli/index.ts`): catches anything from commands, prints
   `error: <redacted message>` (stack only with `--verbose`), calls
   `process.exit(exitCodeFor(err))`. Unexpected errors print a bug-report hint and exit 10.
6. Unit tests: exit-code mapping for every class; redactor replaces a fake
   `FAKE_API_KEY=sk-abcdefgh12345678` value embedded in a thrown Error message and in a
   nested stack string; short values (<8 chars) are not redacted (avoid over-matching);
   logger level gating; stdout/stderr separation.

## Acceptance Criteria

- [ ] Exit-code table in code is byte-equal to DESIGN §13 semantics; a failing command
      never exits 0.
- [ ] A provider error whose message embeds the API key prints with `***` in place of the
      key (test proves it).
- [ ] All log output goes to stderr; nothing but reports/JSON goes to stdout.
- [ ] `exitCodeFor` returns 10 for non-AppError values (including strings/undefined).

## Validation

- `npx vitest run tests/unit/util` green.
- Manual: `FAKE_API_KEY=sk-test1234567890 node dist/cli.js` with a forced throw containing
  the key → stderr shows `***`.

## Dependencies

01

## Non-goals

Structured JSON logging, log files, i18n of error messages, telemetry of any kind.

## Design References

- DESIGN §13 (exit codes), §15.1 (quiet/console split), §16 T-SECRET
