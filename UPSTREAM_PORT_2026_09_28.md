# September 28 upstream port for Barnard

## Decision and baseline

Port the independently compatible Coder retry budget and scratch-file guidance.
Defer the three workspace-backed cloud providers until the core and Harness
workspace migrations can be validated together against Barnard's application.
Do not replace local tools with remote providers as part of this backport.

Inspected fork: `orderofmagnitude1/pydantic-ai-harness`, `main` at
`176aff9e5411f8688baec26dbddf4e282a44fb7b` (September 20).
That commit is an ancestor of the original Harness upstream at
`e1732b29a6e5980fc6b291ccb60ea7696ad90ecd`, with zero fork-only commits.
No Barnard-named code was found in the published tree. This does not establish
the contents of unpublished branches, a local checkout, or Barnard's application.

The fork already has `pydantic_ai_harness.modal_sandbox.ModalSandbox`, including
its session, toolset, lifecycle tests, and optional `modal` extra. Searches that
reported no `ModalSandbox` symbol missed this implementation. E2B and Sprites
providers are absent.

## Exact upstream changes

| Commit | Behavior | Disposition |
| --- | --- | --- |
| [`82e79d34f79e003342f09b0ce783642f915c2975`](https://github.com/pydantic/pydantic-ai/commit/82e79d34f79e003342f09b0ce783642f915c2975) | Configurable file-tool retries; Coder defaults to five; scratch-file instructions. | Adapt implementation and tests to this fork's local-path API. |
| [`82bdbf5b2ec4c36fceaaac012ebed2bbca75c376`](https://github.com/pydantic/pydantic-ai/commit/82bdbf5b2ec4c36fceaaac012ebed2bbca75c376) | Replace Modal's dedicated tools/session with a workspace backend. | Defer: incompatible APIs and lifecycle semantics. Keep the existing Modal implementation. |
| [`a0b9050f695cfb43d8c7d2807840a82e947a0656`](https://github.com/pydantic/pydantic-ai/commit/a0b9050f695cfb43d8c7d2807840a82e947a0656) | Add persistent Sprites workspace, backend, SDK extra and conformance tests. | Defer with the workspace migration. |
| [`8fd47479527a9ba6426d10b74e10768b76eea0fb`](https://github.com/pydantic/pydantic-ai/commit/8fd47479527a9ba6426d10b74e10768b76eea0fb) | Add E2B workspace, backend, SDK extra and conformance tests. | Defer with the workspace migration. |

Upstream moved Harness under `src/pydantic_ai_harness/` and tests under
`tests/harness/`. The backport maps these to this repository's existing paths;
it does not import the monorepo structure, lockfile, release configuration,
documentation navigation, or upstream CI/runner changes.

## Compatible backport

- Add keyword-only `FileSystem(max_retries=None)` and pass it to the existing
  `FunctionToolset(max_retries=...)` API. Standalone FileSystem retains the
  agent's retry budget by default. Zero is supported.
- Give Coder's five file tools a budget of five consecutive errors per tool.
  A successful call resets that tool's counter; the sixth consecutive error
  fails. This is bounded correction, not unlimited retries or a new runtime.
- Preserve `Coder(workspace, ...)`, the 60,000-character read cap, content-hash
  defaults, `cwd`, protected patterns, shell environment filtering, and custom
  instruction appending. Do not copy upstream's newer 50,000-character default
  or workspace-only constructor.
- Add the upstream scratch-file paragraph to existing instructions. It asks
  the model to clean up its own scratch files; it installs no cleanup code and
  grants no deletion permission. Requested artifacts remain requested changes.
- Unlike upstream's `isinstance(value, int)` validation, reject booleans as
  retry budgets as well as negative and non-integer values.
- Use core's existing retry handling and telemetry; add no new spans or payloads.

The tests adapt upstream's denied-write and exhaustion cases to local roots,
and additionally exercise successful reset, stale-edit recovery, zero retries,
read-only wrappers, invalid inputs, unchanged outside/protected files, and
preservation of appended instructions.

## Why the provider patches cannot be copied alone

The locked dependency is `pydantic-ai-slim==2.44.0`. Inspection of the installed
package confirmed that it has no `pydantic_ai.workspaces`,
`AbstractCapability.get_workspace`, or `RunContext.workspace`. Its
`FunctionToolset` does already accept `max_retries`, so the retry port requires
no dependency change.

All three providers depend on two earlier September 28 commits:

1. [`e6df9225accea77f7e61a20a568164b19c49dcfe`](https://github.com/pydantic/pydantic-ai/commit/e6df9225accea77f7e61a20a568164b19c49dcfe)
   adds core workspaces, references, lifecycle hooks, message persistence,
   durable execution, UI plumbing, and the shared conformance suite.
2. [`ef13bc118077307771a6455c86d8fabeb93363a8`](https://github.com/pydantic/pydantic-ai/commit/ef13bc118077307771a6455c86d8fabeb93363a8)
   migrates Harness tools and supporting capabilities to `ctx.workspace`,
   including filesystem, shell, memory, repository context, skills,
   subagents, output stores, and CLI integration (180 changed files).

Adding a provider without migrating consumers would either fail imports on
the locked core or leave file/shell operations using the host. Reimplementing
workspace primitives inside Harness would violate the repository's core
boundary and split execution semantics.

There are also application-visible changes:

- Modal stops registering its own tools, removes `ModalSandboxSession`, and
  moves file/output/command settings onto FileSystem and Shell. Several old
  constructor arguments become warnings and are ignored; the old
  `max_command_timeout` ceiling has no direct equivalent. Newly owned
  sandboxes no longer terminate at run end: the default lifetime is 24 hours,
  with no idle timeout unless configured. This changes cleanup and billing.
- Sprites persists files until explicit deletion; ending a run closes an
  owned client, not the Sprite. It adds `sprites-py>=0.7.0,<0.8` and needs
  reference ownership and cleanup across resumed and durable runs.
- E2B defaults to internet access and a one-hour lifetime that pauses rather
  than deletes the sandbox. Attach resumes it. It adds `e2b>=2.48.0` and explicit
  destruction remains the application's responsibility.
- Modal's new backend raises its SDK floor from `modal>=1.5.0` to `>=1.5.2`.

These providers could benefit autonomous development and synthetic-data work.
They are not evidence that remote execution is an approved destination for
Barnard patient data. Keep them upstream-only for this backport; reconsider
them as opt-in execution backends after the migration, not as default clinical
workspaces.

## Follow-up acceptance conditions

Use a core version verified to contain the workspace prerequisites, migrate
all workspace consumers together, and preserve Barnard's application contracts
with regression tests. Keep provider SDKs optional. Exercise upstream's fake
backend, conformance, cancellation, process cleanup, unavailable-reference,
and durable replay suites. Add application tests for reference authorization,
patient/workspace isolation, allowed data destinations, network policy,
bounded cost/lifetime, and explicit destruction of retained environments.
Only then run credentialed provider smoke tests with synthetic data.

This port does not update Barnard's application dependency pin or certify its
clinical workflows; that application is outside the inspected repository.

## Local verification

With the existing lockfile, Python 3.14.7 and core 2.44.0:

```bash
uv sync --locked --group lint
uv run --no-sync ruff format --check .
uv run --no-sync ruff check .
PYRIGHT_PYTHON_IGNORE_WARNINGS=1 uv run --no-sync pyright \
  pydantic_ai_harness/coder pydantic_ai_harness/filesystem \
  tests/coder/test_tools.py tests/filesystem/test_editing_options.py
uv run --no-sync pytest -p no:cacheprovider tests/filesystem tests/coder \
  tests/test_coder.py tests/test_docs_parity.py tests/test_doc_snippets.py -q
```

Results: Ruff formatting/lint passed; targeted Pyright reported zero errors
and warnings; pytest reported 1,401 passed and 25 skipped. The skipped tests
are not counted as verified. This fork's Coder integration test is a
deterministic tool exercise, not a provider cassette replay. No paid model,
cloud sandbox, patient-data, or live Barnard application tests were run.
