---
name: tstyche-migrate-from-tsd
description: Use when replacing tsd with TSTyche in an existing project, including migrating `.test-d.ts` or `.test-d.tsx` files, tsd assertions or configuration, package scripts, CI, or programmatic tsd integrations.
---

# Migrate from tsd to TSTyche

Use this skill for a repository migration from `tsd` to TSTyche. Preserve the intent and effective compiler environment of the existing type tests; do not treat the migration as a global text replacement.

## Workflow

1. Inventory the repository before editing. Find the package manager and workspace boundaries, the installed `tsd` and TypeScript versions, `tsd` imports, `.test-d.ts` and `.test-d.tsx` files, the `package.json#tsd` block, CLI flags, scripts, CI jobs, and any default `tsd()` or `formatter` imports. Run the existing type-test command and record whether it passes.
2. Read the project migration reference. Check the selected TSTyche release's Node and TypeScript requirements against every affected package. Preserve the current package manager, script names, layout, and compiler behavior unless TSTyche requires a change.
3. Add TSTyche while keeping `tsd` available for comparison. Rename type tests to `.tst.ts` or `.tst.tsx`, or intentionally configure `testFileMatch`. Give type tests a TSConfig that represents the old effective compiler options instead of accepting a silent change of defaults.
4. Read the assertion mapping reference and convert every imported helper. Direct relation assertions can be translated systematically; inspect each `expectError` in context and choose the matcher that expresses the invalid operation. Do not remove an assertion that has no direct equivalent.
5. Verify discovery and configuration with `tstyche --listFiles` and `tstyche --showConfig`. Run one migrated file first, then the full type-test command and the repository's normal validation. Compare the result with the recorded `tsd` baseline and investigate new diagnostics rather than suppressing them wholesale.
6. Remove `tsd`, its package configuration, and obsolete files only after TSTyche passes and a final search finds no remaining imports, invocations, CLI flags, or unresolved assertions. Update the lockfile with the repository's package manager.

## Migration boundaries

- TSTyche loads the project's TypeScript version and can expose differences hidden by `tsd`'s bundled compiler. Treat changed diagnostics as compatibility work, not automatic test rewrites.
- TSTyche rejects accidental `any` and `never` sources by default. Keep those protections unless the project intentionally tests those types and the assertion cannot express that intent explicitly.
- If the repository imports the default `tsd` function, `formatter`, or consumes tsd diagnostics, also load `tstyche-programmatic-api`. There is no safe one-line API replacement; redesign that integration around TSTyche's public runner, events, results, or tag entrypoint.
- Stop and report any `expectDeprecated`, `expectNotDeprecated`, `printType`, or `expectDocCommentIncludes` use that still needs coverage. A migration is not complete while such behavior is silently missing.

## Read as needed

- Assertion conversions, error cases, and unsupported helpers: [references/assertion-mapping.md](references/assertion-mapping.md)
- Dependencies, filenames, configuration, scripts, CI, and staged cleanup: [references/project-migration.md](references/project-migration.md)
