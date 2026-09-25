# Embedded `Runner`, `Cli`, and watcher failures

Use this file when an embedded TSTyche integration rejects, exits unexpectedly, leaks event handlers, or stays alive after the host test completes. The live guide is https://tstyche.org/guides/programmatic-usage.

## Resolution contracts

- `tstyche/tag` (default export) is a tagged-template function that resolves on a successful run and rejects with an `Error` whose message names the failing test file. Catch the rejection at the host boundary and preserve stdout/stderr if diagnostics are part of the assertion.
- `Runner.run` resolves after dispatching events, not after each test passes or fails. A passing event stream does not mean the overall run succeeded. Inspect events or wait for the runner's terminal event when an exit-code contract is required.
- `Cli.run` returns an exit-code-like result suitable for process-style integration. Use it when a programmatic equivalent of the CLI exit code is required.
- Call `Config.resolve(...)` rather than hand-writing a `ResolvedConfig`. The resolution logic handles defaults, `--config` files, environment variables, and CLI overrides; a partial resolved config bypasses precedence.

## Cancellation

- `Runner.run(files, cancellationToken)` accepts a `CancellationToken`. Without a token, the runner cannot be cancelled mid-flight.
- Pass an `AbortSignal`-shaped token and propagate cancellation from the host test. Without it, a watcher will keep the process alive after the host completes.
- For `tstyche/tag`, cancellation is propagated via the parent cancellation signal. Wrap the call in `AbortController` if the host needs a hard timeout.

## Watcher leakage

- A watcher observes the filesystem through events. The async iterator returned by the watch run can keep Node's event loop busy after the host test is done.
- Always tear down watchers explicitly. Either pass a `CancellationToken` and call `.cancel()` after the host test, or use the runner's terminal event to detach handlers.
- Test for leak: run the embedded integration in a fixture, end the test, and assert that the process exits with `code 0` and no pending event handlers. A timer or file watcher still alive at that point is the leak.

## Reporter cleanup

- A custom reporter is a default-exported class with a constructor receiving `ResolvedConfig` and an `on([event, payload])` method. Reporters register with the runner; the runner removes them on completion.
- Reporters that hold external state (open files, intervals, network connections) must clean it up in their `on` handling of the run's terminal event. Otherwise they leak between runs.
- Test module resolution for a package, a relative file, and a missing spec. The barrel-neighbor resolution differs across Node and bundlers; pin the resolution the integration actually uses.

## Standard fixtures

- Set `--root` to the fixture project path; `--config` is required only when the config lives outside that root.
- Set `--quiet` when the host test should own output, and `--reporters` explicitly when parsing reporter output (built-ins are `dot`, `list`, `summary`).
- Set `--tsconfig` and `--target` when host-project discovery could select the wrong compiler settings or TypeScript version.
- Use an explicit temporary `TSTYCHE_STORE_PATH` for tests that fetch TypeScript versions. Two concurrent fixtures sharing one store race for the cache; one isolated store per fixture avoids collisions.

## Failure paths

- Configuration errors (`config:error`, `select:error`) emit as typed events. Handle them before any logical run event; otherwise the host assumes the run started.
- A missing or unreadable config file rejects the run. Catch the rejection and assert on the message rather than the exit code.
- A reporter that ignores the typed event union will accept payloads it cannot read. Narrow the event name before reading any field.
- `Runner.run` on an empty `files` list succeeds without doing anything. Pass at least one `string`, `URL`, or `FileLocation` to make the run meaningful.

## Symptom-to-cause quick map

| Symptom | Likely cause |
| --- | --- |
| Host test never exits | watch iterator alive, no cancellation token |
| `Runner.run` resolves but tests failed | forgetting that success means dispatch, not pass |
| Custom reporter ignored | default-export shape wrong, or `reporters` not pointing to the module |
| Exit code from `tstyche/tag` is `0` on test failure | unwrapping the rejection at a different layer; catch at the host boundary |
| Concurrent fixtures corrupt the store | shared `TSTYCHE_STORE_PATH`, missing per-fixture override |
