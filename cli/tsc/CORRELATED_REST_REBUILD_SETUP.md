# Correlated object-rest rebuild — deno vendored-compiler patch

## What this is

A checker feature that recognizes an **identity reconstruction** of a destructured
value and re-derives its original (still-correlated) type. For a discriminated union
`S`, the two expressions below are identical at runtime, but stock TypeScript infers
different types:

```ts
// Form A — correlation preserved
items.map((item) => [item.code, item] as const)

// Form B — destructure then rebuild; correlation LOST upstream, PRESERVED with this patch
items.map(({ code, ...rest }) => [code, { ...rest, code }] as const)
```

When an object literal is exactly `{ ...rest, p1, p2, … }` assembled from the
unmodified pieces of a single `const { p1, p2, …, ...rest } = s`, the literal is
provably equal to `s`, so its type is re-derived as `typeof s` instead of the
decorrelated spread result (in which the discriminant is widened across every union
member). The feature is **unconditional** (no compiler option / no opt-out).

## How the vendored compiler was regenerated

deno's `deno check` (default path) runs the bundled isolate compiler in
`cli/tsc/00_typescript.js` — a build of the **denoland/TypeScript** fork carrying
~60 deno-specific compiler patches (node/deno globals separation, `data:` URL
handling, custom libs, `dom.extras`, …). A plain microsoft/TypeScript build cannot be
dropped in without losing those patches.

Rather than rebuild the fork, the feature was applied **surgically** to the existing
vendored blob, leaving every deno patch untouched. Two edits inside the
`createTypeChecker` closure of `cli/tsc/00_typescript.js`:

1. Added `function getCorrelatedRestRebuildType(node, spreadResultType)` immediately
   before `function checkObjectLiteral(...)`. It is a direct port of the same function
   in microsoft/TypeScript `src/compiler/checker.ts` (numeric `SyntaxKind` constants:
   PropertyAssignment 304, ShorthandPropertyAssignment 305, SpreadAssignment 306,
   ComputedPropertyName 168, ObjectBindingPattern 207; `CheckMode.Normal` 0). All
   helpers it calls already exist in the blob by name (`getResolvedSymbol`,
   `getTypeOfSymbolAtLocation`, `isSymbolAssigned`, `getTextOfPropertyName`,
   `getTypeForBindingElementParent`, `isTypeAssignableTo`, `isBindingElement`,
   `skipParentheses`).

2. In `checkObjectLiteral`, the spread-result return was changed from
   `return mapType(spread, …)` to capture the result and consult the recognizer:
   `const rebuiltType = getCorrelatedRestRebuildType(node, spreadResultType);`
   `return rebuiltType || spreadResultType;`

Because the feature is unconditional, **no change to `99_main_compiler.js` is needed**
(no compiler option to force on). The blob is embedded into the binary at build time
(`cli/build.rs` zstd-compresses it; `cli/tsc/mod.rs` `include_bytes!`s the artifact),
so the binary must be rebuilt after editing the blob:

```
cargo build --bin deno
```

## Soundness

The recognizer fires only on a provable identity rebuild: every value is a bare
identifier resolving to an element of one object binding pattern, none reassigned or
narrowed since binding, the literal covers the pattern's rest plus every non-rest
element exactly (no override, extra, rename, default, computed key, or partial
coverage), and a final `isTypeAssignableTo(source, spreadResult)` backstop rejects
lossy sources (non-spreadable members). Any deviation falls back to today's behavior.

## Verifying end-to-end (default path, no flags)

```
./target/debug/deno check canary.ts      # clean — rebuilt value typed as the source union
```

`canary.ts`:

```ts
type Item =
  | { readonly code: "C"; readonly name: "circle" }
  | { readonly code: "R"; readonly name: "square" };
declare const item: Item;
const { code, ...rest } = item;
const rebuilt = { ...rest, code };
const a: Item = rebuilt;            // ok (the direction broken upstream)
const b: typeof rebuilt = item;     // ok
if (rebuilt.code === "C") { const n: "circle" = rebuilt.name; }  // ok
```

Before the patch this errored `TS2322: '{ code: "C" | "R"; … }' is not assignable to
type 'Item'`; after the patch `deno check` reports `Check file://…/canary.ts` and
exits 0. Non-identity rebuilds (override / extra / rename / partial / narrowed /
reassigned) still error, and deno's own globals patches are intact (`setTimeout`
typing unchanged vs stock deno).
