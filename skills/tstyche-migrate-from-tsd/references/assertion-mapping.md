# tsd assertion mapping

Use this reference after inventorying all imports from `tsd`. The source expression or type under test belongs on the left side of a TSTyche assertion. Import `expect` from `tstyche` and preserve aliases only when they improve clarity.

## Direct mappings

| tsd | TSTyche |
| --- | --- |
| `expectType<T>(value)` | `expect(value).type.toBe<T>()` |
| `expectNotType<T>(value)` | `expect(value).type.not.toBe<T>()` |
| `expectAssignable<T>(value)` | `expect(value).type.toBeAssignableTo<T>()` |
| `expectNotAssignable<T>(value)` | `expect(value).type.not.toBeAssignableTo<T>()` |
| `expectNever(value)` | `expect(value).type.toBe<never>()` |

For a type-only assertion, use `expect<Source>().type...` instead of manufacturing a runtime expression. Keep assignability direction unchanged: `expectAssignable<T>(value)` says the type of `value` is assignable **to** `T`. The equivalent target-first spelling, `expect<T>().type.toBeAssignableFrom(value)`, is valid but usually makes the subject less obvious.

TSTyche's `toBe` performs structural type equality. Run the converted assertion rather than assuming every edge case has identical behavior to tsd's compiler-based identity check, especially for overloaded, conditional, inferred, `any`, and `never` types.

## Convert `expectError` by intent

`expectError(expression)` is context-sensitive. Do not mechanically wrap the same expression with a deprecated error matcher. Identify the operation that should be rejected and use the narrowest ability matcher:

```ts
// tsd
expectError(parse(123));

// TSTyche
expect(parse).type.not.toBeCallableWith(123);
```

| Invalid operation | Preferred TSTyche form |
| --- | --- |
| Function or method call | `expect(callable).type.not.toBeCallableWith(...args)` |
| Constructor call | `expect(Constructor).type.not.toBeConstructableWith(...args)` |
| Generic type arguments | `expect<Generic<_>>().type.not.toBeInstantiableWith<[...args]>()`, importing `_` as a type from `tstyche` |
| JSX component props | `expect(Component).type.not.toAcceptProps(props)` in a `.tsx` file |
| Decorator application | Apply `@(expect(decorator).type.not.toBeApplicable)` to the declaration under test |
| Missing property | `expect<Subject>().type.not.toHaveProperty(key)` when property existence is the contract |

Pair a negated ability assertion with a nearby positive case when practical, changing only the input that should be rejected. This proves the subject itself is usable and documents the boundary under test.

When the diagnostic exists only inside a larger expression that an ability matcher cannot reproduce, keep the expression and place `@ts-expect-error` immediately above it. Include the expected diagnostic text so TSTyche can verify the suppressed error; use `...` only for unstable portions. Do not use a directive that could also pass because an import or symbol is missing. See https://tstyche.org/guides/expect-errors.

`toRaiseError` is deprecated. It may help inspect an old test temporarily, but it is not a completed migration target.

## Helpers without direct equivalents

The following tsd helpers have no direct TSTyche assertion equivalent:

- `expectDeprecated`
- `expectNotDeprecated`
- `expectDocCommentIncludes`
- `printType`

Do not delete or approximate them with unrelated type assertions. Record each occurrence and explain the lost contract. For deprecation and documentation-comment checks, retain a separate supported check or ask the user which replacement they accept. Treat `printType` as diagnostic scaffolding: remove it only if the user does not require equivalent diagnostic output.

## Review traps

- Preserve `.tsx` when migrating JSX tests and make sure the selected TSConfig has the intended `jsx` option.
- Preserve top-level `await`, module specifiers, type-only imports, and literal narrowing; changing these can change the inferred type independently of the framework migration.
- TSTyche rejects unexpected inferred `any` and `never` by default. Fix the source of an accidental type rather than disabling the protection globally. Explicit `any` or `never` targets remain valid when the test intentionally names them.
- Search for namespace imports, renamed imports, re-exports, and wrapper helpers around tsd; named-import replacement alone is not a complete inventory.
