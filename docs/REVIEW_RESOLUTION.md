# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #1. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTNkCj86QAySo — execute local npm tarballs with npm exec

Finding: Passing a local tgz as a positional npx executable argument attempts to execute the archive rather than install its package command.

Normative resolution:
- The local package specification must be passed through npm exec's package option: npm exec --package "$spec" -- i18n-agent ...
- The command must resolve the executable from the package metadata/bin contract, not from the tarball pathname itself.
- Quoting, argument forwarding, and non-zero package-resolution failures must be preserved in the CLI contract.

Focused verification before resolving this thread:
- Invoke the Action/CLI with a local tgz specification and assert the package is installed into the isolated execution context and its declared i18n-agent command is selected.
- Include spaces and shell metacharacters in the temporary path and assert they are treated as data.
- Test package-resolution and executable-missing failures and require a non-success terminal result without executing the archive as a program.

## PRRT_kwDOTNkCj86QAySu — workflow trigger scanning is syntax-scoped

Finding: A literal grep for pull_request_target can flag documentation text and miss the distinction between a workflow's parsed on keys and unrelated values.

Normative resolution:
- Parse each workflow YAML and inspect only trigger keys under the workflow's on mapping, including the documented pull_request_target key.
- Exclude docs, comments, examples, and non-workflow files from this policy check unless the policy explicitly includes them.
- The check must handle YAML's boolean-like on key representation consistently and report the workflow path and parsed trigger.

Focused verification before resolving this thread:
- Use fixtures containing the token in docs/comments, as a real trigger key, and as an unrelated value; assert only the policy violation is reported.
- Test quoted/unquoted forms and common YAML parser behavior for the on key.
- Confirm the scan covers every workflow file in the repository and returns a deterministic result.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.