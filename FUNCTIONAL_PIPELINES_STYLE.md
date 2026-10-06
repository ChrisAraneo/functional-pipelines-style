# Functional Pipelines Style

These rules apply to every TypeScript file in a repository that adopts this
guide.

**MUST** and **NEVER** mean exactly that. A rule may only be broken where this
guide names the exception; section 15 is the only list of exceptions, and
anything added to it carries a measurement.

This guide is repo-agnostic. Section 16 lists the five blanks a repository fills
in before the guide is usable, and the lint rules that enforce it mechanically.
A repository may add its own rules in a sibling document; it may not relax the
ones here.

## 1. The single expression rule

> **A function body is a single expression: one pipeline that composes small
> named steps. A body contains no statement-level control flow.**0

NEVER write `if`, `else`, `switch`, `?:`, `for`, `while`, `for…of`, `forEach`,
`try`, `catch`, `throw`, `let`, `var`, a reassignment, or a staircase of
intermediate `const`s. Imperative code of any kind is forbidden in a body. A
body reads as one `chain(…)` or `flow(…)` that flows a value from input to
output. The exported function names and orders the steps; it does no logic of
its own.

Four tools do all of the work:

| Tool                           | Import from            | Use it for                                                        |
| ------------------------------ | ---------------------- | ----------------------------------------------------------------- |
| `match(x).with(…)…`            | `ts-pattern`           | every branch, whether it has two arms or twenty                   |
| `chain(value).thru(…).value()` | a local lodash wrapper | the backbone of a body: flow a value you hold through named steps |
| `flow(a, b, c)`                | `lodash-es`            | naming a function that _is_ the composition of existing ones      |
| `tryCatch(tryer, catcher)`     | `ramda`                | turning a call that can throw into a value                        |

When you see the construct on the left, write the one on the right instead. This
table is the checklist for a refactor.

| Imperative construct                                          | Replacement                                                             |
| ------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `if (x) … else …`                                             | `match(x).with(…).otherwise(…)`                                         |
| `cond ? a : b`                                                | `match(x).with(…).otherwise(…)`                                         |
| `if / else if / else`, `switch`                               | `match(x).with(…).with(…).exhaustive()` or `.otherwise(…)`              |
| `instanceof` / `typeof` ladder                                | `match(x).with(instanceOf(Error), …)` after `const { instanceOf } = P;` |
| `for` / `while` / `forEach` that accumulates                  | `map`, `filter`, `reduce`, `times`, `range`, or `chain().thru()`        |
| `try { … } catch (error) { … }`                               | `tryCatch(() => …, (error) => fallback)`                                |
| `throw new Error(…)`                                          | return the fallback from the `tryCatch` catcher                         |
| `let acc = …; acc = …`                                        | thread the value through `.thru()` or `flow` steps                      |
| `const a = …; const b = f(a);`                                | `chain(a).thru(f).thru(g).value()`                                      |
| `if (x == null) return` guards                                | thread `X \| undefined`, short-circuit with `?.` or lodash `get`        |
| `() => undefined`                                             | lodash `noop`                                                           |
| `.length`, `items.map(…)`                                     | `size(…)`, `map(items, …)`                                              |
| `_.map(…)`, `import _ from 'lodash'`                          | named imports from `lodash-es` (section 10.2)                           |
| `Object.keys/values/entries`, `JSON.parse(JSON.stringify(x))` | `keys`/`values`/`entries`, `cloneDeep(x)` (section 10.3)                |
| `typeof x === 'string'`, `Array.isArray(x)`                   | `isString(x)`, `isArray(x)` (section 10.3)                              |
| `for await`, `await` inside a loop                            | `Promise.all(map(…))`, or a `reduce` over a promise (section 10.10)     |
| `Math.random()`, `Date.now()`                                 | a generator or clock passed in as an argument (section 7)               |

## 2. Files and folders

1. **One export per file.** Name the file after its export, in kebab-case:
   `findEligibleItems` → `find-eligible-items.ts`, `ProductCategory` →
   `product-category.ts`.
2. **Named exports only.** NEVER use `export default`.
3. **Every file that exports a function has a spec beside it** with the same
   base name: `find-eligible-items.spec.ts`. Files in `internal/` get specs
   too.
4. **Each feature gets its own folder:**

   ```text
   checkout/
   ├── place-order.ts                 entry point, the only file callers import
   ├── place-order.spec.ts
   └── internal/                      private to checkout/
       ├── find-eligible-items.ts
       ├── find-eligible-items.spec.ts
       └── …
   catalog/
   ├── get-categories.ts              a folder may have several entry points
   ├── pick-featured-products.ts
   ├── slice-for-page.ts
   ├── internal/
   └── types/                         types that callers outside catalog/ use
       ├── category.ts
       └── product-group.ts
   ```

5. **Files at the root of a feature folder are its entry points.** Code outside
   the folder imports only those. NEVER import from another folder's
   `internal/`.
6. **Types get their own file.** A type used only inside a feature goes in
   `internal/<name>.ts`. A type that callers use goes in `types/<name>.ts`.
   NEVER declare a type in a file that holds a function.
7. **Shared helpers move up a level.** A helper that two features need in the
   same form lives in their parent folder. If each feature needs a slightly
   different version, each keeps its own copy in its `internal/`. NEVER add a
   flag parameter just so two features can share one function.
8. **Constants.** A tuning value that several files or tests need goes in the
   nearest `consts.ts`. A value only one file needs is a non-exported
   `UPPER_SNAKE_CASE` constant at the top of that file:
   `const RETRY_ROUNDS = 2;`. Name every number whose meaning is not plain from
   the line it sits on.
9. **Freeze exported arrays and objects** and type them `readonly`:

   ```ts
   export const RETRY_DELAYS: readonly number[] = Object.freeze([200, 1_000]);
   ```

## 3. Functions

1. **Arrow functions only**, assigned to a `const`. NEVER use the `function`
   keyword, a class, or `this`.
2. **Single-expression bodies.** Write `(items) => …`, not
   `(items) => { return …; }`.
3. **Name a value mid-expression with `chain`**, never with a local:

   ```ts
   chain(filter(findNeighbours(grid, row, column), isCountable))
     .thru((found) => maxBy(uniq(found), (item) => countOf(found, item)))
     .thru((commonest) => commonest ?? FALLBACK)
     .value();
   ```

4. **Carry extra state forward as an object**, adding one property per step.
   This is the replacement for a staircase of `const`s; NEVER introduce a `let`.

   ```ts
   export const parsePayload = (input: Input): Output =>
     chain({ raw: getField(input) })
       .thru(({ raw }) => ({ raw, parsed: parse(raw) }))
       .thru(({ raw, parsed }) => ({ parsed, enriched: enrich(parsed, raw) }))
       .thru(({ enriched }) => format(enriched))
       .value();
   ```

5. **`chain` versus `flow`.** Use `chain` when you hold a value and want to flow
   it through steps. Use `flow` when you want to name a new function that _is_
   the composition of existing ones:

   ```ts
   export const normalizeName = flow(trim, toLower, capitalize);
   ```

   They nest: a `.thru()` step may be a `flow(…)`, and a `flow` step may be a
   `chain(…).value()`.

6. **Extract each meaningful step into its own named function** so the chain
   reads as a sentence. Reuse an existing step before writing a new one.
7. **Block bodies are a narrow exception.** A block body is allowed only when a
   function needs several named values that are each read more than once, and
   then it holds nothing but `const` declarations and one `return`. Prefer
   `chain`.
8. **No mutation.** NEVER reassign a name or change an argument. Return new
   arrays and objects. Copy with `[...items]` or `{ ...record }`. Lodash
   `reverse` changes the array it is given, so copy first:
   `reverse([...items])`.
9. **No loops.** Use lodash `map`, `flatMap`, `filter`, `reduce`, `times` and
   `range`. For the one kind of measured exception, see section 15.
10. **Argument order is fixed per domain and never varies.** Pick the order once
    — the subject the function works on first, then its coordinates, then the
    injected effects last — and keep it identical in arguments and in object
    literals: `isSurface(grid, row, column)`, `{ row, column, value }`.
11. **Entry points and helpers take plain arguments. Pipeline steps take one
    object** (section 6).
12. **Return types.** Helpers MUST declare their return type, including
    `| undefined` variants. Pipeline steps and entry points MUST NOT, because
    their types are inferred and the next step reads them through
    `ReturnType<typeof …>`.

    A helper:

    ```ts
    export const getRank = (categories: Category[], index: number): number =>
      countIn(take(categories, index), categories[index]);
    ```

    A step:

    ```ts
    export const pickCandidate = ({
      grid,
      candidates,
    }: ReturnType<typeof sortCandidates>) => ({
      grid,
      candidate: head(candidates),
    });
    ```

## 4. Branching

1. **`match` from `ts-pattern` does every branch.** There is no arm-count
   threshold: a two-way `if` and a ten-way `switch` are both a `match`.
2. **Unions:** one `.with()` per member, ending in `.exhaustive()`:

   ```ts
   match(layout)
     .with('HORIZONTAL', () => groupIntoColumns(grid, candidates))
     .with('VERTICAL', () => groupIntoRows(grid, candidates))
     .exhaustive();
   ```

3. **Booleans and other open values:** `.with(true, …)` or `.with(value, …)`,
   ending in `.otherwise(…)`.
4. **Destructure every pattern helper off `P` at the top of the file.** NEVER
   write `P.` in a pattern position — not `P.nullish`, `P.union`, `P.not`,
   `P.string`, `P.number`, `P.instanceOf`, `P.when`:

   ```ts
   const { nullish } = P;

   …
     .with(nullish, () => [])
     .otherwise((found) => expand(found, FILLER, FILL_HEIGHT))
   ```

5. **Conditions:** use `.when(predicate, handler)`, or a `P` pattern such as
   `number.lt(0)` after `const { number } = P;`.
6. **Name special values before you match on them:** `const NOT_FOUND = -1;`,
   then `.with(NOT_FOUND, () => 0)`.
7. **Keep unions typed.** When handlers return members of a string union, put
   the type on the handler: `.with(true, (): Layout => 'VERTICAL')`.
8. **Shape and type patterns** replace object checks and `instanceof` ladders:

   ```ts
   const { number, array } = P;

   export const getStatusLabel = (state: RequestState): string =>
     match(state)
       .with({ kind: 'loading' }, () => 'Loading…')
       .with({ kind: 'error', code: number }, ({ code }) => `Failed (${code})`)
       .with(
         { kind: 'ready', items: array() },
         ({ items }) => `${size(items)} items`,
       )
       .exhaustive();
   ```

9. **Small cases need no `match`.** Use `??` for a fallback
   (`RETRY_DELAYS[attempt - 1] ?? NO_DELAY`), `?.` for reads that may fall off
   the structure (`rows[row]?.[column]`), and `&&`, `||` and `!` inside
   predicates.
10. **Keep each branch to one call.** When a branch needs more, move it into its
    own helper file and call that.

## 5. Failure, absence and effects

1. **`tryCatch` from `ramda` replaces `try`/`catch` and `throw`.**
   `tryCatch(tryer, catcher)` returns a function that runs `tryer`; when it
   throws, `catcher` receives the error and returns the value the caller
   continues with. There is no second channel to fold.
2. **Annotate the `const`** you assign a `tryCatch` to, because its inference is
   weak, and give the catcher a fallback of the same type as the tryer's result:

   ```ts
   export const loadSettings: () => Settings = tryCatch(
     () => parseSettings(localStorage.getItem(STORAGE_KEY)),
     () => createDefaultSettings(),
   );
   ```

3. **Normalize an unknown error with `match`**, in its own named helper:

   ```ts
   const { instanceOf, string } = P;

   const getErrorMessage = (error: unknown): string =>
     match(error)
       .with(instanceOf(Error), (error) => error.message)
       .with(string, (text) => text)
       .otherwise(() => 'Unknown error occurred');
   ```

4. **Async failure folds on the promise.** `tryCatch` is synchronous and cannot
   catch a rejection. Pass both handlers to `then`:

   ```ts
   export const loadUser = (id: string): Promise<User | undefined> =>
     fetchUser(id).then(normalizeUser, (error) => {
       showToast(getErrorMessage(error));

       return undefined;
     });
   ```

5. **Thread absence as `X | undefined`** and let later steps short-circuit with
   `?.` or lodash `get`. NEVER scatter `x == null` guards:

   ```ts
   export const getPrimaryEmail = (
     user: User | undefined,
   ): string | undefined =>
     chain(user)
       .thru((user) => user?.emails)
       .thru((emails) => emails?.[0])
       .value();
   ```

6. **Side effects live at the edge**, in the outermost handler, never mid-chain.
   When a chain step must run an effect and pass its value on, use a `tapEffect`
   helper of your own:

   ```ts
   export const tapEffect = <T>(value: T, effect: (value: T) => void): T =>
     match(effect(value)).otherwise(() => value);
   ```

   NEVER write `.thru(() => void expr)` or `.thru(() => null)` — both collapse
   the lodash wrapper type to `never` and `.value()` disappears. When a step
   really must yield `null`, type it: `.thru((): Shader | null => null)`.

## 6. Pipelines

A change made in more than one step is a pipeline. Every pipeline has this
shape.

1. **The entry point** takes plain arguments, packs them into one object and
   runs the steps with `flow` from `lodash-es`. It does nothing else.

   ```ts
   export const clearExpired = (records: Record[], now: number) =>
     flow(
       getExpiryWindow,
       findExpiredRecords,
       shuffleExpiredRecords,
       pickExpiredRecords,
       createRemovals,
       applyRemovals,
     )({ records, now });
   ```

2. **One step per file**, in `internal/`. A step takes one object, destructures
   it in its parameter list and returns a new object.
3. **Step input types.** The first step writes its input type inline. Every
   later step uses the return type of the step before it, imported with
   `import type`:

   ```ts
   import { head } from 'lodash-es';
   import type { sortCandidates } from './sort-candidates';

   export const pickCandidate = ({
     records,
     candidates,
   }: ReturnType<typeof sortCandidates>) => ({
     records,
     candidate: head(candidates),
   });
   ```

4. **Pass on only what later steps read.** Pass each field on unchanged: the
   same object, not a copy. Drop a field as soon as no later step reads it.
5. **The last step returns the result itself**, not an object.
6. **Change data through collected edits.** Describe each change as a value in a
   `create…` step, then apply them all in the last step. NEVER write into the
   structure you were given; build the new one from the old one and the edits.
7. **One pipeline per operation, not per variant.** Variants go through the same
   steps; branch on the variant inside the steps that differ, never in the entry
   point.
8. **Name the standard steps the same way everywhere.** A repository fixes one
   order for the recurring shape — `get…` the parameters, `find…` the
   candidates, `shuffle…`/`sort…` them, `pick…` the winners, `create…` the
   edits, `apply…` them — and every pipeline follows it.

## 7. Determinism and injected effects

1. **NEVER call an ambient source of truth inside a function that computes.**
   No `Math.random`, `Date.now`, `new Date()`, `crypto.randomUUID`,
   `performance.now`, `localStorage`, `process.env`, `fetch`, or logging. Take
   what you need as an argument. Routing one of these through lodash does not
   launder it: lodash `random`, `now` and `uniqueId` are ambient in exactly the
   same way, and this section outranks the lodash-first rule (section 10.5).
2. **The injected effect is the last argument**, named for what it is (`random`,
   `now`, `fetchJson`). It is passed down unchanged; NEVER build a second one
   part-way down.
3. **One generator per run, created at the outermost entry point**, from an
   explicit seed, so the same seed MUST always give the same output:

   ```ts
   export const generate = (name: string) =>
     chain(createRandom(name)).thru(buildWorld).value();
   ```

4. **A pipeline carries the generator as a field** until the last step that
   draws from it, then drops it.
5. **The order of the draws is part of the output.** Each draw moves the
   generator on, so one extra or missing draw anywhere changes everything drawn
   after it. Treat a reordering as a behaviour change and re-record the
   expectations.
6. **Shuffle through the injected generator**:
   `sortBy(items, () => random())`.
7. **IO and rendering happen at the edge** — in the handler, the component, the
   CLI entry — and never inside a step.

## 8. Naming

1. **Case.** `camelCase` for functions and values, `PascalCase` for types,
   `UPPER_SNAKE_CASE` for module constants and for string-union members
   (`'HORIZONTAL'`), `kebab-case` for files and folders.
2. **Steps carry the feature's name**, so no two steps in the repository share a
   name: `findOrderCandidates`, never `findCandidates`. A helper that serves one
   step may use a shorter name that fits its job.
3. **Use words, not letters.** Callback parameters say what they hold:
   `(candidate) => candidate.column`, `(row, index) => …`. NEVER use
   one-letter names. Name a parameter you do not use `_`.
4. **One word per concept, repository-wide.** Keep the domain glossary in a
   table (section 16) and use exactly those words — never a synonym, never an
   abbreviation, never `x`/`y` where the domain says `row`/`column`. Record
   which way each counter counts (`pageNumber` from 1, `index` from 0) and
   convert at the point of use: `RATES[pageNumber - 1]`.
5. **Keep a function-name index** (for example `docs/FUNCTION_NAMES.md`) in
   alphabetical order, ignoring case. Check it before you name a function and
   reuse its words. Add every name you add and remove every name you delete.

## 9. Types

1. **`interface` for object shapes; `type` for unions, aliases, records and
   function types:**

   ```ts
   export interface Spot {
     row: number;
     column: number;
   }

   export type Category = 'NORMAL' | 'HARD' | 'BONUS';

   export type CategoryGroups = Record<Category, Item[]>;
   ```

2. **String unions for fixed sets of values.** NEVER use `enum`.
3. **NEVER use** `any`, non-null assertions (`!`), `@ts-ignore`,
   `@ts-expect-error` or `as` casts in source files.
4. **Reuse types instead of writing them out again:** `ReturnType<typeof step>`
   for step inputs, `ReturnType<typeof createRandom>` for an injected generator,
   `Parameters<typeof fn>` for a wrapper.
5. **Import types as types:** `import type { Item } from './types/item';`, or
   `import { type Item, EMPTY_ITEM } from './types/item';` when one line brings
   in both. Turn on `verbatimModuleSyntax` so the compiler checks it.
6. **Run the typecheck the way the repository is wired.** In a monorepo without
   a root project reference, a root `tsc -b` can report nothing while packages
   are broken; check each package.

## 10. Libraries

lodash is the single vocabulary for iteration, collections, objects, strings,
numbers, cloning and type checks. A native construct appears only where this
section names it.

1. **`lodash-es` for everything it covers**, called as functions and imported by
   name: `map(items, …)`, `size(items)`, `head(items)`. NEVER call array methods
   (`items.map(…)`) and NEVER read `.length`; use `size()`. Reach for a lodash
   helper before writing one of your own.
2. **Named imports from `lodash-es`, and nothing else.**
   - One `import { … } from 'lodash-es';` per file, kept in sync with the
     helpers actually used.
   - NEVER `import _ from 'lodash-es'`, NEVER `import * as _ from 'lodash-es'`,
     NEVER a `_.`-prefixed call. The `_` namespace does not appear in source.
     (A bare `_` as the name of an unused parameter is a different thing and
     stays allowed — section 8.3.)
   - NEVER the CommonJS `lodash` package, and NEVER `require('lodash')`. If a
     file still has either, replace it with named `lodash-es` imports and leave
     nothing behind.
   - `lodash-es` ships no types: `@types/lodash-es` MUST be a dev dependency, or
     the named imports do not type-check.
3. **The replacement table.** Each native construct on the left is replaced by
   the lodash helper on the right.

   | Native                                                                                     | lodash                                                                                                       |
   | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
   | `for`, `while`, `for…of`, `for…in`, `.forEach`                                             | `map`, `filter`, `reduce`, `times`, `range` — never `forEach` (rule 4)                                       |
   | `.map`, `.filter`, `.reduce`, `.find`, `.some`, `.every`, `.includes`, `.flatMap`, `.sort` | `map`, `filter`, `reduce`, `find`, `some`, `every`, `includes`, `flatMap`, `sortBy`                          |
   | `a && a.b && a.b.c`                                                                        | `get(a, 'b.c')`                                                                                              |
   | `Object.prototype.hasOwnProperty.call(obj, 'id')`                                          | `has(obj, 'id')`                                                                                             |
   | `const { password, ...safe } = user`                                                       | `omit(user, ['password'])`, `pick(user, [...])`                                                              |
   | `Object.keys` / `.values` / `.entries`                                                     | `keys` / `values` / `entries` (alias `toPairs`)                                                              |
   | `str.trim()`, `str.charAt(0).toUpperCase() + …`                                            | `trim(str)`, `upperFirst(str)`                                                                               |
   | hand-rolled case conversion                                                                | `camelCase`, `kebabCase`, `snakeCase`, `startCase`, `capitalize`, `lowerCase`, `pad`, `truncate`, `repeat`   |
   | `Math.round`, `Math.floor`, `Math.ceil`                                                    | `round`, `floor`, `ceil`                                                                                     |
   | `Math.max(...nums)`, `Math.min(...nums)`                                                   | `max(nums)`, `min(nums)`; also `clamp`, `inRange`, `sum`                                                     |
   | `JSON.parse(JSON.stringify(obj))`                                                          | `cloneDeep(obj)`                                                                                             |
   | `Object.assign({}, a, b)`, `{ ...a, ...b }`                                                | `assign({}, a, b)`; `merge({}, a, b)` when the merge is deep                                                 |
   | `Array.isArray(x)`, `typeof x === 'string'`                                                | `isArray(x)`, `isString(x)`                                                                                  |
   | `x === null \|\| x === undefined`                                                          | `isNil(x)`; also `isEmpty`, `isUndefined`, `isNumber`, `isBoolean`, `isFunction`, `isPlainObject`, `isEqual` |
   | an accumulator loop building a lookup                                                      | `keyBy`, `groupBy`, `countBy`, `uniqBy`, `compact`                                                           |
   | `Date.now()`                                                                               | `now()` — at the edge only (rule 5)                                                                          |

4. **`forEach` is banned, in both forms.** `items.forEach(…)` and
   `forEach(items, …)` are statement-level loops that exist only for their side
   effects, and a body holds no side effects (section 5.6). A loop becomes
   `map`, `filter`, `reduce`, `times` or `range`; an early exit becomes `find`,
   `some`, `every`, `takeWhile` or `dropWhile`. This is the one place where the
   lodash-first rule does not extend to the obvious helper.
5. **`random` and `now` are ambient, and section 7 outranks lodash.** NEVER call
   lodash `random`, `now` or `uniqueId` inside a function that computes: a
   seeded generator or a clock comes in as an argument. Use `now()` only where
   the clock itself is created, at the edge.
6. **Mutating helpers are banned; the fresh-target forms are not.** `merge` and
   `assign` write into their first argument, so the first argument MUST be a new
   literal: `assign({}, a, b)` and `merge({}, a, b)` are fine,
   `assign(target, source)` and `merge(target, source)` are not. NEVER use
   `set`, `unset`, `pull`, `remove`, `fill` or `reverse` on a value you were
   given; build the new value instead (`{ ...record, [key]: value }`,
   `reverse([...items])`). `cloneDeep` replaces every hand-rolled deep copy.
7. **Prefer the arrow over the property shorthand.** `map(users, 'name')`
   type-checks but infers loosely; `map(users, (user) => user.name)` keeps the
   tighter type. Use the shorthand only where the inferred type is still exact,
   and add explicit type arguments where inference degrades.
8. **lodash type guards narrow**, so they compose with `match`:
   `.when(isString, …)`. Inside a `match`, prefer the `P` pattern (`string`,
   `number`, `array()`); use a lodash guard where you need it outside a pattern
   position, or inside `.when`. After each swap, confirm the narrowed type still
   satisfies the code downstream.
9. **Dates are not lodash's job.** Beyond `now()`, lodash has no date surface.
   NEVER invent a lodash date helper or force arithmetic through `add`/
   `subtract` where real date semantics are meant; take a date library
   (`date-fns`, `dayjs`) and say so.
10. **Async stays out of the collection helpers.** No lodash helper awaits
    anything: `forEach` and `map` ignore returned promises. Run independent work
    with `Promise.all(map(items, loadOne))`; sequence dependent work by folding
    with `reduce` over a promise
    (`reduce(items, (done, item) => done.then(() => handle(item)), Promise.resolve())`).
    NEVER write `for await` or an `await` inside a loop in a body.
11. **`chain` must come from a local wrapper, NEVER straight from `lodash-es`.**
    lodash-es attaches the wrapper's methods through a `mixin(lodash, lodash)`
    side effect that bundlers tree-shake away, so in a production build
    `chain(x)` returns a wrapper with no methods at all and fails only on the
    deployed site — neither the test suite nor the dev server catches it.
    Re-export it once, off the lodash default export, and import `chain` from
    that module everywhere:

    ```ts
    import * as lodashModule from 'lodash-es';

    const lodash = (lodashModule as unknown as { default: typeof lodashModule })
      .default;

    export const { chain } = lodash;
    ```

    This wrapper module is the one place a namespace import of lodash is
    allowed. End every chain with `.value()`.

12. **`flow`** from `lodash-es` runs pipelines and composes steps.
13. **`match` and `P`** from `ts-pattern` do all branching.
14. **`tryCatch`** from `ramda` captures every throw. Import ramda functions by
    name and call them bare: NEVER `import * as R from 'ramda'`, NEVER
    `R.tryCatch`.
15. **`noop`** from `lodash-es` is the only way to write a no-op thunk. NEVER
    write `() => undefined`.
16. **Arithmetic.** Use lodash `floor`, `ceil`, `round`, `sum`, `clamp` and
    `inRange`, and lodash `max` and `min` on arrays. `Math` keeps only the
    scalar cases lodash does not cover: `Math.abs`, and `Math.min`/`Math.max`
    on two single numbers.
17. **Sorting.** `sortBy` is stable: it keeps tied items in their old order, and
    tests depend on it. Sort from high to low by negating the key
    (`(candidate) => -candidate.column`). Break ties with a list of keys:

    ```ts
    sortBy(candidates, [
      (candidate) => candidate.row,
      (candidate) => Math.abs(candidate.column - getMiddleColumn(grid)),
    ]);
    ```

    `sortBy` returns a new array; `reverse` does not, so copy first.

18. **One library per job.** Do not add a second collection, pattern-matching or
    date library beside the ones above; extend the local wrapper module instead.
19. **Imports across package boundaries** use the repository's path aliases.
    Imports inside a package use relative paths, spelled the way its neighbours
    spell them (with or without the file extension — pick one per repository).

## 11. Layout and comments

1. **Prettier decides the formatting.** Keep its defaults close to: single
   quotes, semicolons, trailing commas, two-space indents, 80 columns. Never
   hand-format.
2. **Import order:** package imports first, then relative imports. Sort each
   group by path.
3. **Source files have no blank line between the two import groups. Spec files
   have exactly one.**
4. **One blank line between top-level statements.** The `P` destructuring
   (`const { nullish } = P;`) goes right after the imports.
5. **No comments.** Names and tests explain the code. The only comments allowed
   are tool directives, such as `// eslint-disable-next-line`, and the note on
   each exception in section 15.

## 12. Tests

1. **One runner, imported by name.** With Vitest, `import { describe, expect, it } from 'vitest';`.
   NEVER use mocks, spies, `beforeEach`, `afterEach`, snapshots, `.only`,
   `.skip` or `it.each`. A body that takes its effects as arguments (section 7)
   needs no mock: pass a stub function.
2. **One `describe` per file, named exactly after the function:**
   `describe('pickCandidate', …)`. Only a spec that checks behaviour across
   several features uses a plain-English title.
3. **Titles read `should … when …`**, in plain everyday English about the
   domain, not the code. Call the function "it". Say "the basket", "the spots"
   and "the edits", not `items`, `cells` and `patches`:
   - `'should keep the basket the same when it collects the edits'`
   - `'should give no spots when the page holds no slot'`
   - `'should charge the reduced price when the order clears the free-shipping threshold'`
4. **Test each step on its own.** Every step in `internal/` has its own spec.
   The entry point's spec checks the feature as a whole.
5. **Every spec covers:**
   - the main behaviour, and the empty case (an empty collection, no
     candidates);
   - for a pipeline step: one test for each field it passes on, using `toBe` to
     show the same object came back;
   - for an entry point or an `apply…` step: that the input it was given is
     unchanged (`'should not change the old basket when …'`);
   - for anything that draws from an injected generator: the same seed gives the
     same result, and another seed gives a different one. Give each call its own
     `createRandom(seed)`, because a shared generator moves on with every draw.
     A step that only passes the generator on may share one `RANDOM` constant
     across its tests;
   - for anything that branches on a variant: every member of the union;
   - for anything wrapped in `tryCatch`: that the failing path returns the
     fallback.
6. **Fixtures.**
   - Shared read-only fixtures are `UPPER_SNAKE_CASE` constants:
     `const BASKET: Item[] = [ITEM_APPLE];`
   - When a test checks that the input is left alone, build the fixture with a
     factory so every call makes a fresh one: `createBasket()`.
   - Write a bulky fixture in compact literal notation and expand it through a
     legend, so the test reads as the thing it describes:

     ```ts
     const CELLS: Record<string, Cell> = {
       '#': CELL_WALL,
       o: CELL_COIN,
       P: CELL_EXIT,
     };

     const createGrid = (rows: string[]): Cell[][] =>
       map(rows, (row) => map([...row], (cell) => CELLS[cell] ?? CELL_EMPTY));
     ```

   - Wrap the call in a helper that returns only the field under test:

     ```ts
     const findCandidates = (rows: string[], layout: Layout) =>
       findGridCandidates({ grid: createGrid(rows), layout }).candidates;
     ```

7. **Assertions.** `toEqual` for values. `toBe` for primitives and to show the
   same object came back. `toBeCloseTo` for fractions. When you check many cases
   inside `times(…)`, pass a message so a failure names the case:
   ``expect(cleared, `round ${round}`)``.
8. **Inside a test**, separate the setup, the call and the `expect`s with blank
   lines.
9. **A slow test** that needs more than the default time passes a timeout as the
   third argument of `it`: `120000`.
10. **Specs may also:** cast with `as unknown as` to build a fixture that is hard
    to type, write into a fixture they have just made, collect calls in a local
    array (`offsets.push(offset)`), and declare small local types. Every other
    rule in this guide applies to specs too.
11. **Know the command that actually runs them.** In a monorepo whose task
    runner is misconfigured, a green root command can mean nothing ran; run the
    runner inside the package.

## 13. Turning imperative code into a pipeline

**Read the whole file first** and note its existing imports. Then work in this
order, one construct at a time. The result MUST behave exactly as the code did
before: same outputs, same short-circuiting, same mutation or non-mutation.
Change the style, not the logic — do not rename variables or restructure
anything the refactor does not touch.

1. **Name the steps.** Read the body top to bottom, find each distinct
   transformation and extract it into a small named `const` arrow, in its own
   file when the surrounding feature has a folder.
2. **Replace every branch** — `if`/`else`, `?:`, `switch`,
   `instanceof`/`typeof` ladder, whatever its arm count — with a `match`.
3. **Wrap every thrower** in `tryCatch`, with the catcher returning the
   fallback. Fold async rejections with `promise.then(onOk, onError)`. Move the
   side effect to the outermost edge.
4. **Inject the ambient effects** the body reaches for — generator, clock,
   storage, network — as arguments (section 7).
5. **Thread absence** as `X | undefined`, short-circuiting with `?.` or lodash
   `get`, and delete the null guards.
6. **Kill the locals.** Collapse the `const` staircase and every `let` into one
   `chain(…).thru(…).value()` or `flow(…)`, carrying extra context by adding a
   property per step.
7. **Sweep the natives into lodash**, row by row down the table in section 10.3,
   checking each swap for the semantics the original had: an early exit becomes
   `find`/`some`/`every`/`takeWhile`, never a `forEach` that returns `false`; an
   `await` in a loop becomes `Promise.all` or a `reduce` over a promise, never a
   `forEach` that drops the promise; a deep `merge` keeps its fresh `{}` target.
8. **Fix the imports.** One named `import { … } from 'lodash-es'`, in sync with
   the helpers now used; `chain` from the local wrapper; every `'lodash'`
   import, `require('lodash')` and `_.`-prefixed call gone. Confirm no `_.`
   call remains.
9. **Declare the return type** on every helper, including `| undefined`.
10. **Update the function-name index** (section 8.5) with every name added or
    removed.
11. **Run the typecheck, the linter and the specs** for the package, and add the
    missing specs from section 12. A refactor is not done until all three pass.

## 14. Remove these on sight

- Any imperative code in a body: a `const` staircase, a `for`/`while` loop, a
  `let` accumulator, a reassignment.
- An `if`, `else`, `switch` or `?:` that is not a `match`.
- A `try`, `catch` or `throw` anywhere in a body.
- `import * as R from 'ramda'`, or an `R.tryCatch` call.
- `import { chain } from 'lodash-es'` instead of the local wrapper.
- An `import` of the CommonJS `lodash` package, a `require('lodash')`, an
  `import _ from 'lodash-es'`, or any `_.`-prefixed call.
- A `forEach`, in either form — `items.forEach(…)` or `forEach(items, …)`.
- A mutating lodash call on a value that was passed in: `set`, `unset`, `pull`,
  `remove`, `fill`, `reverse`, or `merge`/`assign` whose first argument is not a
  fresh literal.
- `Object.keys`/`values`/`entries`, `JSON.parse(JSON.stringify(x))`,
  `Array.isArray(x)`, `typeof x === '…'`, or a hand-rolled string case
  conversion, where the lodash helper belongs.
- `map(users, 'name')` where the arrow form keeps a tighter type.
- A `for await`, or an `await` inside a loop, in a body.
- An invented lodash date helper, or date arithmetic forced through
  `add`/`subtract`.
- A `P.*` used inline in a pattern instead of destructured at the top of the
  file.
- An `instanceof` or `typeof` ladder.
- A `.length` read, or an array method (`.map`, `.filter`, `.forEach`) where a
  lodash function belongs.
- Scattered `x == null` or `x === undefined` guards.
- A hand-written `() => undefined`.
- A side effect (toast, log, draw, write) buried mid-pipeline instead of at the
  outermost handler.
- An exported function that does real logic itself instead of only composing
  named steps.
- `Math.random`, `Date.now`, `process.env` or a second generator inside a
  computing function.
- A comment that explains the code instead of a better name or a test.

## 15. Measured exceptions

Imperative code survives in exactly two shapes, and only where a measurement
says it must. Each one carries a comment naming the measurement.

1. **A hot numeric kernel** whose mutation never leaves the function, entered
   often enough that one closure call per iteration dominates the cost. Keep the
   loop; everything that calls it stays a pipeline.
2. **A single mutable state cell** inside a generator or iterator, where the
   state _is_ the semantics (a PRNG, a cursor). Isolate the cell so the bodies
   around it stay single expressions.

**Measure before assuming.** Figures from one profiled TypeScript codebase, as
an order of magnitude: a `ts-pattern` `match` costs roughly 85ns per call and a
`chain().thru().value()` roughly 135ns — cheap enough for per-item use in
anything a user waits on. What actually hurt was a closure inside a
million-iteration numeric kernel: the pipeline forms of one blur accumulator
measured 93ms against 263ms per run. Record your own numbers in the comment, and
add nothing to this list without them.

## 16. Adopting this guide in a new repository

Fill in these five blanks, then the guide is complete for that repository:

1. **The dependencies.** `lodash-es`, `ts-pattern` and `ramda` in
   `dependencies`; `@types/lodash-es` in `devDependencies`, without which the
   named lodash imports do not type-check.
2. **The `chain` wrapper module** (section 10.11) and its import path.
3. **The domain glossary** (section 8.4): one table of the project's words, each
   with its meaning, the type it names, and which way its counters count.
4. **Where constants live** (section 2.8): the path of the nearest `consts.ts`
   per package.
5. **The real commands** for formatting, linting, typechecking and testing,
   including any monorepo quirk that makes a root command lie (sections 9.6,
   11.1, 12.11).

Enforce what a linter can. These `no-restricted-syntax` selectors are verified
to fire on the constructs they name, under ESLint flat config:

```js
'no-restricted-syntax': [
  'error',
  { selector: 'IfStatement', message: 'Use match() from ts-pattern.' },
  { selector: 'SwitchStatement', message: 'Use match() from ts-pattern.' },
  { selector: 'ConditionalExpression', message: 'Use match() from ts-pattern.' },
  { selector: 'ForStatement', message: 'Use map/filter/reduce.' },
  { selector: 'ForOfStatement', message: 'Use map/filter/reduce.' },
  { selector: 'ForInStatement', message: 'Use map/filter/reduce.' },
  { selector: 'WhileStatement', message: 'Use map/filter/reduce.' },
  { selector: 'DoWhileStatement', message: 'Use map/filter/reduce.' },
  { selector: 'TryStatement', message: 'Use tryCatch from ramda.' },
  { selector: 'ThrowStatement', message: 'Return a fallback value instead.' },
  { selector: 'VariableDeclaration[kind="let"]', message: 'Thread the value.' },
  { selector: 'VariableDeclaration[kind="var"]', message: 'Thread the value.' },
  { selector: 'FunctionDeclaration', message: 'Use an arrow const.' },
  { selector: 'ClassDeclaration', message: 'Use functions.' },
  { selector: 'TSEnumDeclaration', message: 'Use a string union.' },
  {
    selector: "CallExpression[callee.property.name='forEach']",
    message: 'Use map/filter/reduce.',
  },
  { selector: "CallExpression[callee.name='forEach']", message: 'Use map/filter/reduce.' },
  { selector: "ImportDeclaration[source.value='lodash']", message: 'Use lodash-es.' },
  {
    selector: "ImportDeclaration[source.value=/^lodash/] ImportDefaultSpecifier",
    message: 'Use named imports.',
  },
  {
    selector: "ImportDeclaration[source.value=/^lodash/] ImportNamespaceSpecifier",
    message: 'Use named imports (the chain wrapper is the one exception).',
  },
  { selector: "MemberExpression[object.name='_']", message: 'No _ namespace.' },
  {
    selector:
      "ImportDeclaration[source.value='lodash-es'] ImportSpecifier[imported.name='chain']",
    message: 'Import chain from the local wrapper.',
  },
],
```

The `_` rule targets `_.` calls through `MemberExpression`, not the identifier,
so naming an unused parameter `_` (section 8.3) stays clean. Exempt the `chain`
wrapper module itself from the namespace-import rule — it is the one file that
needs `import * as lodashModule`.

Add to that: `prefer-const`, `no-param-reassign`,
`@typescript-eslint/no-explicit-any`,
`@typescript-eslint/no-non-null-assertion`, and
`@typescript-eslint/consistent-type-imports`.

In an existing codebase these rules light up everywhere at once. Turn them on as
warnings, fix per folder, and only then raise them to errors — and never relax a
rule to make a file pass. Bring the file in line instead.
