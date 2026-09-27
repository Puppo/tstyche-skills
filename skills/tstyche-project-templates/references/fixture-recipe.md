# Reproducible fixture recipe

Use this file when authoring the smallest TSTyche layout that another piece of code can target. A fixture is reproducible when the host code can run the same way regardless of where it is invoked from; it is minimal when removing any file breaks the host contract. The live references are https://tstyche.org/project/file-structure and https://tstyche.org/guides/programmatic-usage.

## Minimum layout

```text
fixture/
  package.json
  tstyche.json
  tsconfig.json
  __typetests__/
    smoke.tst.ts
```

Five files. Every other file is a deliberate addition.

## `package.json`

```json
{
  "name": "fixture",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "tstyche"
  },
  "devDependencies": {
    "tstyche": "*",
    "typescript": "*"
  }
}
```

Pin `tstyche` to the version under test. Pin `typescript` only when the fixture must use a specific compiler version; otherwise the host's `tstyche --target` selection wins.

## `tstyche.json`

```json
{
  "$schema": "./node_modules/tstyche/schemas/config.json",
  "target": "*"
}
```

The schema reference gives editor validation. `target: "*"` lets the host pass `--target` without a config conflict. Keep `testFileMatch` at the default and explicitly verify with `--showConfig` once `--root` and `--config` are wired up.

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "noEmit": true,
    "strict": true,
    "types": []
  },
  "include": ["./__typetests__/**/*"],
  "exclude": []
}
```

`types: []` keeps ambient `@types/*` packages from changing the fixture's environment. `include` whitelists only the test directory. `exclude: []` prevents a workspace-level `exclude` from leaking in.

## `__typetests__/smoke.tst.ts`

```ts
import { expect, test } from "tstyche";

test("smoke: package is reachable", () => {
  expect<string>().type.toBe<string>();
});
```

One assertion that always passes under any version TSTyche supports. Its purpose is to prove the fixture is wired correctly, not to test the host. A fixture that asserts something interesting is no longer minimal.

## Verification

- `tstyche --root ./fixture --target 5.8` from the host succeeds and exits non-zero on any failure.
- `tstyche --showConfig --root ./fixture` prints the resolved options. Run the fixture to see the `uses TypeScript ... with ...` line and confirm the selected TSConfig.
- The host's `tstyche/tag` import resolves from the host module, while `--root` selects the fixture project. Verify the imported TSTyche version from the host and install the fixture dependencies before running it.

## Embedded host wiring

```ts
import tstyche from "tstyche/tag";

const fixtureRoot = new URL("./fixture", import.meta.url);
await tstyche`--quiet --root ${fixtureRoot} --target 5.8`;
```

`--quiet` lets the host own the output. Absolute `--root` avoids CWD drift between runs. `tstyche/tag` resolves on success and rejects on non-zero exit (including assertion failures), so wrap the call in a `try/catch` whose assertion checks both the rejection and the stderr capture when diagnostics matter.

For `Runner`-based integrations, parse options and the fixture config before constructing the runner:

```ts
import { fileURLToPath } from "node:url";
import { Config, Runner } from "tstyche/api";

const root = fileURLToPath(new URL("./fixture/", import.meta.url));
const { commandLineOptions, pathMatch } = await Config.parseCommandLine([
  "--quiet", "--root", root, "--target", "5.8",
]);
const { configFileOptions } = await Config.parseConfigFile(
  commandLineOptions.config,
  commandLineOptions.root,
);
const resolved = Config.resolve({ commandLineOptions, configFileOptions, pathMatch });
await new Runner(resolved).run([new URL("./fixture/__typetests__/smoke.tst.ts", import.meta.url)]);
```

`Config.resolve` merges the parsed options with defaults into a `ResolvedConfig`; it does not parse arguments or read the file itself. `Runner.run` does not return an exit code, so use `Cli.run([...])` or `tstyche/tag` when the host needs pass/fail status.

## Common mistakes

- Adding `.gitignore`, `README.md`, or `LICENSE` files that the host test shouldn't depend on. They're noise at minimum scope.
- Putting fixture tests in a shared `__tests__` directory. Production compilation can include them accidentally; a dedicated `__typetests__` is the recommended shape.
- Including the fixture in the host's own `tsconfig.json`. Treatment as production code breaks the boundary.
- Running the host with `npx tstyche` from a workspace that already has its own `tstyche`. Pin the fixture's install via `package.json` and `npm install` inside the fixture.
- Sharing one fixture across matrix builders that each spawn their own child. Concurrent children race on the store; pass `TSTYCHE_STORE_PATH=/tmp/fixture-store-${pid}` so each child gets its own.
