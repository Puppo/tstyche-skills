# Project migration from tsd

Use this reference for repository structure, configuration, commands, and cleanup. The current TSTyche references are https://tstyche.org/project/file-structure, https://tstyche.org/reference/config-file, and https://tstyche.org/reference/command-line.

## Inventory and compatibility

Inspect every affected workspace rather than assuming the repository root owns type testing. Record:

- the package manager, lockfile, workspace boundaries, Node engine, local TypeScript version, and `tsd` version;
- all `.test-d.ts` and `.test-d.tsx` files and any custom test directory;
- imports, re-exports, wrapper helpers, `package.json#tsd`, CLI flags, package scripts, CI commands, and documentation commands;
- default `tsd()` calls, `formatter`, or code consuming tsd diagnostics.

Run the existing tsd command before changing files. A failing baseline is not migration fallout; preserve the result in the handoff.

Before installing, check the chosen TSTyche package's `engines` and `peerDependencies` and the live prerequisites. TSTyche uses a compatible local `typescript` package when one is installed, unlike tsd's bundled `@tsd/typescript`. Do not remove tsd if meeting TSTyche's Node or TypeScript requirement would require an unapproved platform upgrade.

## Files and compiler configuration

The preferred conversion is:

| tsd project shape | TSTyche project shape |
| --- | --- |
| `name.test-d.ts` | `name.tst.ts` |
| `name.test-d.tsx` | `name.tst.tsx` |
| default `test-d/` directory | existing dedicated directory, or `__typetests__/` / `typetests/` if the project chooses to rename it |
| `tsd.directory` | renamed files discovered by defaults, or an explicit `testFileMatch` glob |
| `tsd.compilerOptions` | compiler options in a dedicated type-test TSConfig |

Prefer renaming files over keeping the declaration-file suffix. Preserve the existing directory unless moving it has a concrete benefit; the migration does not authorize a broader layout refactor. Renaming the directory is optional: the default `testFileMatch` finds `*.tst.*` files anywhere, while the `__typetests__/` and `typetests/` patterns only add discovery for files named `*.test.*`.

Give a dedicated type-test directory an isolated TSConfig that extends the project config, sets `noEmit`, and explicitly includes the tests. Prefer `strict: true` and `types: []`, but preserve an intentional old strictness setting and list any ambient type packages that are part of the tested contract. Carry across other intentional options from `tsd.compilerOptions`, including `lib`, `jsx`, module settings, path aliases, and decorators. Confirm the file is actually included: when TSTyche's default `findup` mode finds no TSConfig that includes a test, it falls back to baseline options.

tsd forced `skipLibCheck: false`, and TSTyche defaults to `checkDeclarationFiles: true`, which also forces `skipLibCheck: false`, so declaration-file checking matches tsd's behavior out of the box. Treat an explicit `checkDeclarationFiles: false` as an intentional opt-out (typically for performance), not as an automatic consequence of honoring the project TSConfig.

Do not mechanically copy tsd's historical compiler defaults. They reflect older TypeScript behavior and may be unsupported or counterproductive with the selected current TypeScript version. For a migration whose goal is current TypeScript compatibility, start with TSTyche's [baseline compiler options](https://tstyche.org/project/compiler-options#default-compiler-options) and add a dedicated TSConfig only for project-specific options or intentionally preserved behavior. If exact pre-migration behavior is a requirement, record the effective tsd configuration and reproduce only the options that remain supported by the selected TypeScript version; otherwise treat changed diagnostics as part of the migration. Use `tstyche --showConfig` and the run header to verify the compiler version, effective options, and TSConfig selected for every layout.

## Configuration and CLI mapping

| tsd | TSTyche |
| --- | --- |
| `tsd` package script | keep the script name and replace its command with `tstyche` |
| `tsd.directory` | `testFileMatch`, or default discovery after renaming |
| `tsd.compilerOptions` | dedicated TSConfig selected by discovery, `tsconfig`, or `--tsconfig` |
| `--files` / `-f`, `testFiles` | `testFileMatch` plus positional search strings for focused runs |
| project path / programmatic `cwd` | run in that workspace or use `--root`; pair with `--config` when config lives elsewhere |
| `--typings` / `-t`, `typingsFile` | no direct flag; import the entrypoint or the specific declaration file under test in each test file |

tsd loaded the typings entrypoint automatically and failed when it was missing; TSTyche only checks what a test file imports. A migrated file that imports nothing from the project checks nothing, so make sure every converted test imports the entrypoint or the declaration file under test. Project-wide declaration errors still surface through `checkDeclarationFiles`, which defaults to `true`.

Add `tstyche.json` only for intentional runner overrides. Use its installed schema and keep compiler options in TSConfig. `checkDeclarationFiles`, `checkSuppressedErrors`, `rejectAnyType`, and `rejectNeverType` default to `true`; new failures from those checks need review, not blanket disabling.

Preserve CI behavior and script entrypoints. Replace direct `npx tsd`, `pnpm tsd`, `yarn tsd`, or equivalent calls with the repository's package-manager form of TSTyche. If the project supported multiple TypeScript versions through custom jobs, express that contract with TSTyche `--target` or `target`; do not invent a version matrix when none existed.

## Verification and cleanup

Use a staged cutover:

1. Install TSTyche without removing tsd and update the lockfile normally.
2. Migrate one representative file and run it by positional search string.
3. Run `tstyche --listFiles` and compare the selected files with the inventory; zero tests or a partial list is a failure.
4. Run `tstyche --showConfig` and verify root, globs, TSConfig, TypeScript target, and defaults.
5. Migrate the remaining assertions and run the full type-test script, then the repository's normal checks.
6. Search again for `from "tsd"`, `from 'tsd'`, `require("tsd")`, command invocations, `package.json#tsd`, `.test-d.ts`, `.test-d.tsx`, and unsupported helper names.
7. Remove tsd and stale configuration only when the search is clean and TSTyche passes. Reinstall with the existing package manager if needed to produce a consistent lockfile.

For monorepos, repeat discovery and configuration checks from each script's actual working directory. TSTyche path resolution follows the effective root and config location, so one successful root run does not prove every workspace is configured correctly.

## Programmatic usage

A default import such as `import tsd from "tsd"`, a `formatter` import, or code consuming tsd diagnostic arrays is not a CLI/assertion migration. Load `tstyche-programmatic-api` and redesign the caller around the documented `tstyche/api` runner, events, reporters, and results, or `tstyche/tag` for an embedded CLI-style run. If the companion skill is unavailable, use https://tstyche.org/guides/programmatic-usage and the installed public declarations as the source of truth. Preserve the caller's success, failure, output, cancellation, and working-directory contracts and test them separately before removing tsd.
