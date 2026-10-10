# Functional Pipelines Style

These rules apply to every TypeScript file in a repository that adopts this
guide.

**MUST** and **NEVER** mean exactly that. A rule may only be broken where this
guide names the exception; section 15 is the only list of exceptions, and
anything added to it carries a measurement.

This guide is repo-agnostic. Section 16 lists the four blanks a repository fills
in before the guide is usable, and the lint rules that enforce it mechanically.
A repository may add its own rules in a sibling document; it may not relax the
ones here.

## 1. The single expression rule

**A function body is a single expression: one pipeline that composes small
named steps. A body contains no statement-level control flow.**

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
   `filterEligibleItems` → `filter-eligible-items.ts`, `ProductCategory` →
   `product-category.ts`.
2. **Named exports only.** NEVER use `export default`.
3. **Every file that exports a function has a spec beside it** with the same
   base name: `filter-eligible-items.spec.ts`. Files in `internal/` get specs
   too.
4. **Each feature gets its own folder:**

   ```text
   checkout/
   ├── create-order.ts                entry point, the only file callers import
   ├── create-order.spec.ts
   └── internal/                      private to checkout/
       ├── filter-eligible-items.ts
       ├── filter-eligible-items.spec.ts
       └── …
   catalog/
   ├── get-categories.ts              a folder may have several entry points
   ├── filter-featured-products.ts
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
   chain(filter(getNeighbours(grid, row, column), isCountable))
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
       .thru(({ raw, parsed }) => ({ parsed, patched: patch(parsed, raw) }))
       .thru(({ patched }) => format(patched))
       .value();
   ```

5. **`chain` versus `flow`.** Use `chain` when you hold a value and want to flow
   it through steps. Use `flow` when you want to name a new function that _is_
   the composition of existing ones:

   ```ts
   export const formatName = flow(trim, toLower, capitalize);
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
    export const countEarlierCopies = (
      categories: Category[],
      index: number,
    ): number => countOccurrences(take(categories, index), categories[index]);
    ```

    A step:

    ```ts
    export const getTopCandidate = ({
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
     .with('HORIZONTAL', () => splitIntoColumns(grid, candidates))
     .with('VERTICAL', () => splitIntoRows(grid, candidates))
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
     .otherwise((found) => addFiller(found, FILLER, FILL_HEIGHT))
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

   export const formatStatusLabel = (state: RequestState): string =>
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
   export const readSettings: () => Settings = tryCatch(
     () => parseSettings(localStorage.getItem(STORAGE_KEY)),
     () => createDefaultSettings(),
   );
   ```

3. **Normalize an unknown error with `match`**, in its own named helper:

   ```ts
   const { instanceOf, string } = P;

   const formatErrorMessage = (error: unknown): string =>
     match(error)
       .with(instanceOf(Error), (error) => error.message)
       .with(string, (text) => text)
       .otherwise(() => 'Unknown error occurred');
   ```

4. **Async failure folds on the promise.** `tryCatch` is synchronous and cannot
   catch a rejection. Pass both handlers to `then`:

   ```ts
   export const tryFetchUser = (id: string): Promise<User | undefined> =>
     fetchUser(id).then(toUser, (error) => {
       drawToast(formatErrorMessage(error));

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
   When a chain step must run an effect and pass its value on, use a `runEffect`
   helper of your own:

   ```ts
   export const runEffect = <T>(value: T, effect: (value: T) => void): T =>
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
   export const removeExpiredRecords = (records: Record[], now: number) =>
     flow(
       computeExpiryWindow,
       filterExpiredRecords,
       sortExpiredRecordsRandomly,
       sliceExpiredRecords,
       createRemovals,
       patchRecords,
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

   export const getTopCandidate = ({
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
   `create…` step, then apply them all in the last step, a `patch…` step.
   NEVER write into the structure you were given; build the new one from the
   old one and the edits.
7. **One pipeline per operation, not per variant.** Variants go through the same
   steps; branch on the variant inside the steps that differ, never in the entry
   point.
8. **Name the steps with the verb table** (section 8.6), like every other
   function. In the recurring shape the walk down the table usually ends at
   these verbs, in this order: `compute…` the parameters, `filter…` the
   candidates, `sort…` them (`sort…Randomly` to shuffle), `slice…` the winners,
   `create…` the edits, `patch…` the data with them.

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
   export const createWorld = (name: string) =>
     chain(createRandom(name)).thru(createTerrain).value();
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
   name: `filterOrderCandidates`, never `filterCandidates`. A helper that
   serves one step may use a shorter name that fits its job.
3. **Use words, not letters.** Callback parameters say what they hold:
   `(candidate) => candidate.column`, `(row, index) => …`. NEVER use
   one-letter names. Name a parameter you do not use `_`.
4. **One word per concept, repository-wide.** Before you name a value, search
   the code for the word it already uses for that concept and use exactly that
   word — never a synonym, never an abbreviation, never `x`/`y` where the code
   says `row`/`column`. When the code uses two words for one concept, use the
   more common one. A counter says in its name which way it counts: `…Index`
   counts from 0, `…Number` from 1 (`pageIndex`, `pageNumber`). Convert at the
   point of use: `RATES[pageNumber - 1]`.
5. **Reuse the nouns of existing function names.** The verb comes from rule 6.
   For the rest of the name, search the code for functions that do similar work
   and follow their nouns: `filterOrderCandidates` beside `filterOrderItems`,
   not `filterOrderOptions`.
6. **Every function name starts with a verb from the verb table below.** The
   verb is the first part of the name: the name is either the verb alone
   (`values`, `redraw`) or the verb followed by a word that starts with a
   capital letter (`isEmpty`, `takeRandomServer`). `issueTicket` does not start
   with `is`. Choose the verb with this walk:
   1. Start at the first row of the table and go down one row at a time.
   2. At each row, read **Means** and **Use when**, and decide whether the verb
      suits the operation the function performs.
   3. Use the first verb that suits, and stop. A verb lower in the table never
      wins over a suitable verb above it, even when it seems a closer fit:
      `formatStatusLabel`, not `getStatusLabel`, because **format** comes
      before **get**.
   4. When no row suits, the function does more than one thing. Split it into
      steps until each one suits a row.

   Then apply these rules to the name:
   - NEVER start a name with a word from the **Synonyms** column. Use the verb
     of its row instead: `readSettings`, not `loadSettings`.
   - **try** is a prefix, not a verb of its own, and the walk skips its row.
     Choose the verb with the walk, then put `try` in front when the function
     returns `undefined` instead of failing: `tryFetchUser`.
   - The verb always comes first. Where a row shows a form that puts it
     elsewhere (`sourceToTarget` under **to**) or calls it as a method
     (`user.clone()`), write a plain function that starts with the verb:
     `toFahrenheit(celsius)`, `toUser(dto)`, `cloneUser(user)`.
   - A function in this guide never throws (section 5), so the **throw** row
     never suits.
   - The rule covers every function you name: entry points, steps, helpers and
     spec helpers. Functions imported from a library keep their names (`map`,
     `flow`, `match`, `tryCatch`), and so do parameters that receive an
     injected effect (`random`, `now`, section 7.2).

The verb table. The order of the rows is the order of the walk in rule 8.6.

| Verb | Description | Synonyms (do not use) |
|---|---|---|
| **throw** | **Means:** Signal a failure to the caller by *throwing or raising an exception* (or panicking, in languages without exceptions). The current operation stops and control passes to the nearest error handler; the function does not return normally. Used for helpers that build and throw an error, or that test a condition and throw when it holds (`throwNotFound(id)`, `throwIfCancelled(token)`, `throwIfInvalid(input)`).<br><br>**Use when:** Failures the current code cannot recover from and the caller must deal with: missing resources, broken preconditions, invalid arguments, cancelled or timed-out operations. Name conditional guards `throwIf…`. A function that only builds an error value to return, without throwing it, uses **create** (`createNotFoundError`). Use **try** for variants that return an empty result instead of throwing, **validate** to collect problems without stopping, and **log** to record a problem and carry on.<br><br>**Why this word:** Whether a call can end in an exception is the most important fact about its control flow, and "throw" states it outright. `error`, `fail`, `raise` and `report` leave unclear whether the function throws, returns an error, or only prints one. Language keywords (`throw`, Python `raise`, Go/Rust `panic`) and interface-required names (Go's `Error()` method) are unaffected. | `error`, `raise`, `fail`, `panic`, `bail`, `report`, `complain` |
| **log** | **Means:** Write an *operational or diagnostic message* to a log or trace sink for developers and operators, not as a result for the user (`logRequest`, `logEvent`). It has no effect on program behaviour.<br><br>**Use when:** Application and server logs, request tracing, timing and performance output, verbose or debug messages. The level or channel goes in the name or a parameter (`logDebug`, `log(level, message)`), not in a separate verb. Use **throw** when the problem must stop the operation and **write** for real program output.<br><br>**Why this word:** `trace`, `debug` and `info` are levels of one action. One verb with a level argument keeps log calls searchable and the sinks interchangeable. Logging-library methods (`logger.debug`) keep their names. | `trace`, `debug`, `info`, `verbose` |
| **slice** | **Means:** Take *one contiguous sub-range* of a sequence (array, list, string, buffer, result set) by position, without modifying the original (`slicePage(items, offset, limit)`, `sliceRange`).<br><br>**Use when:** Taking a range of elements or characters by start/end index or offset/limit, and pulling a contiguous part out of a larger whole. Use **split** to break the whole into all its parts, **filter** to select by condition instead of position, and **takeRandom** to select by chance.<br><br>**Why this word:** `slice` is the common term across languages (JavaScript `slice`, Python and Go slicing). `substring`, `substr`, `extract` and `take` mean the same and only add variety. The compound **takeRandom** is a separate favorite and is not affected. Built-ins such as `String.prototype.substring` keep their names. | `extract`, `substring`, `substr`, `take`, `segment` |
| **sort** | **Means:** Put the elements of a collection *in order* according to a key or comparator, including reversed or descending order (`sortByDate`, `sortUsers`, `sortDescending`).<br><br>**Use when:** Any reordering by key, by comparator, ascending or descending: `sortDescending`, not `reverse`, when the goal is an order. Say in the name or documentation whether it sorts in place or returns a sorted copy. Use **match** for the comparator that decides how two elements relate.<br><br>**Why this word:** Every reorder is a sort by some key. One verb keeps ordering logic in one place, and the direction belongs in the name or a parameter rather than a separate verb. Built-ins such as `Array.prototype.reverse` or SQL `ORDER BY` keep their names. | `reverse`, `order`, `arrange`, `rank` |
| **split** | **Means:** Divide one whole into *several parts* at separators or by a rule, returning all the parts (`splitName`, `splitIntoChunks`, `splitPath`).<br><br>**Use when:** Breaking strings at delimiters, paths into segments, lists into pages, batches or chunks, or a set into partitions. Use **slice** to take *one* contiguous piece by position, and **filter** to keep the matching elements.<br><br>**Why this word:** `split` is the universal string term in standard libraries. `divide`, `partition` and `chunk` are the same one-into-many act. | `divide`, `partition`, `chunk`, `separate`, `break` |
| **validate** | **Means:** Test *data or input* (form fields, request bodies, configuration, files, arguments) against a set of rules and *return* whether it is acceptable, or the list of errors found (`validateEmail`, `validateForm`, `validateConfig`).<br><br>**Use when:** Input and configuration validation, schema checks and business-rule checks, where the caller decides what to do with the result. Use **check** for examination that reports or throws by itself.<br><br>**Why this word:** "Validate" says "against rules, with a result", the contract callers depend on. `verify` and `confirm` mean the same, and one word keeps all such entry points findable. | `verify`, `confirm`, `audit`, `sanity` |
| **match** | **Means:** Test how a value *corresponds* to a pattern or to another value: whether it fits a glob, regular expression, route or rule (boolean), or how two values relate (an equality or ordering result) (`matchRoute`, `matchPattern`, `matchVersion`).<br><br>**Use when:** Pattern, route and glob matching, equality and similarity checks, and comparators that report how two values relate. Use **find** to locate a matching item in a collection, and **contains** for membership.<br><br>**Why this word:** Comparing and matching both ask "how do these two line up?". Using `match` for all of them avoids the `compare`/`equals`/`cmp` mix. Language-required forms (Java `equals`/`compareTo`, Python `__eq__`, `localeCompare`) keep their names. | `compare`, `equals`, `cmp`, `fits` |
| **draw** | **Means:** Render something visually: issue drawing or graphics commands to a canvas, screen, image, terminal or GPU (`drawChart`, `drawLine`, `drawSprite`).<br><br>**Use when:** Graphics, charting, game and UI rendering code that draws shapes, images, text or scenes, or sets up graphics state for a draw. Use **redraw** to draw again something that is already on the surface. Low-level graphics-API names (OpenGL/WebGL `uniform…`, `tex…`, `vertex…`) come from the standard and cannot be renamed.<br><br>**Why this word:** "Draw" is the plain word for putting pixels on a surface. Using it for your own rendering code avoids mixing `render`/`paint`/`plot`, which mean the same thing. Framework-required methods (React `render`, Android `onDraw`) keep their names. | `render`, `paint`, `plot`, `uniform`, `tex`, `vertex`, `compressed` |
| **redraw** | **Means:** Draw *again* something that has already been drawn, so the surface shows its current state: the old pixels, characters or GPU output for that area are replaced by a fresh **draw** of the same thing (`redrawChart`, `redrawRow(index)`, `redrawCursor`, `redraw()`). It assumes an earlier draw happened and that the target, such as a canvas, window region, widget, terminal line or scene, is still there; only what is shown changes. A full redraw clears the area and draws it from scratch, and a partial redraw limits the work to a region or the parts that changed (`redrawRegion(rect)`, `redrawDirtyCells`).<br><br>**Use when:** The data behind something on screen changed (new chart values, an edited cell, a moved cursor, a resized window), after the surface was lost or invalidated (context loss, theme switch, device-pixel-ratio change), and in animation or game loops that draw each frame over the previous one. Implement it as clearing the affected area and calling the same **draw** functions again, so first draw and redraw cannot drift apart. When the redraw is not done immediately but marked as needed and done on the next frame or idle tick, say so in the name or a parameter (`redrawOnNextFrame`, `redraw({ deferred: true })`), so callers know nothing has changed on screen yet and repeated calls in one frame are merged. Use **draw** for the first time something appears, **clear** to blank the surface without drawing anything new, and **patch** or **compute** to change or work out the data before it is redrawn.<br><br>**Why this word:** `re` + **draw** says both that the thing is already visible and that the same drawing code runs again, and a search for `draw` finds both halves. `rerender`, `repaint` and `redisplay` name the same act with the synonyms **draw** already replaces, and `refresh` is folded into **patch**, which is about data, not pixels. Framework-required methods (browser `requestAnimationFrame`, Android `invalidate()`, Qt `update()`/`repaint()`, React re-renders triggered by state) keep their names. | `rerender`, `repaint`, `redisplay` |
| **serialize** | **Means:** Convert in-memory structures into a *storage or wire format*, such as JSON, XML, YAML, Protocol Buffers or a binary layout, that can later be read back with **parse** (`serializeOrder`, `serializeSession`).<br><br>**Use when:** API payloads, cache entries, message-queue messages, save files and anything else that is stored or sent and read back later. Use **format** for human-readable display text and **to** for in-memory conversions to another type.<br><br>**Why this word:** "Serialize" is the language-neutral word and pairs cleanly with **parse**. `marshal` and `encode` mean the same thing. Interface-required methods (Go `MarshalJSON`, `JSON.stringify`) keep their names. | `marshal`, `encode`, `stringify`, `pack`, `dump` |
| **hash** | **Means:** Map data of any size to a *fixed-size value*, a hash, digest or checksum, using a hash function such as SHA-256, BLAKE3, xxHash, FNV, CRC32, or a password-hashing function such as Argon2, bcrypt or scrypt (`hashFile(path)`, `hashPassword(password)`, `hashContent(bytes)`, `hashKey(key)`). The same input always gives the same output for a given algorithm and settings (salt, seed, cost), different inputs give different outputs with high probability, and the input cannot be recovered from the hash. The result is returned as bytes, a number or an encoded string (hex, base64) and the input is not modified.<br><br>**Use when:** Content addressing and deduplication, cache keys and ETags, integrity and change detection (has this file changed since the last build?), hash-table keys and `hashCode`-style helpers for your own types, consistent hashing and sharding, and storing passwords. Put what is being hashed after the verb (`hashFile`, `hashRequestBody`), and put the algorithm in a parameter or at the end only when callers must choose it (`hashSha256`, `hash(data, "sha256")`). Keep the two kinds apart in documentation: *fast* hashes for keys, caches and checksums, and *slow, salted* hashes for passwords and other secrets, which must use a password-hashing function, never a plain fast hash. Checking a value against a stored hash is **match** (`matchPassword(password, storedHash)`) and should compare in constant time. Use **serialize** when the data must be read back, **create** for random identifiers or tokens that are not derived from input (`createId`), **compute** for other derived values, and **to** for reversible conversions such as base64 or hex encoding of the same bytes. Encryption and signing are reversible or keyed operations and are not hashing; they follow your cryptography library's names.<br><br>**Why this word:** `hash` is the standard term in every language and library (Java `hashCode`, Python `hash()`/`hashlib`, Go `hash.Hash`, Node `crypto.createHash`), and it tells the reader the result is one-way, deterministic and fixed in size, which `compute` does not. `digest`, `checksum` and `fingerprint` name the result or one use of it, `crc`, `sha` and `md5` name an algorithm, and all of them split one act across several words. Language- and library-required names (`hashCode`, `__hash__`, `GetHashCode`, `Hash()` in Go interfaces, `digest()` on hash objects) keep their names. | `digest`, `checksum`, `fingerprint`, `crc`, `sha`, `md5` |
| **takeRandom** | **Means:** A *compound verb* that chooses one item, or a stated number of items, *at random* from a set of existing candidates, such as an array, set, map's keys, weighted table or deck, and returns what was chosen (`takeRandom(items)`, `takeRandomServer(servers)`, `takeRandomTip(tips, rng)`, `takeRandomMany(questions, 5)`). The result is always one of the candidates the caller passed in or the collection holds; nothing new is made up. The randomness is either *generated earlier* and passed in, as a seeded generator, a random-number source or an already rolled value (`takeRandom(items, rng)`, `takeRandomAt(items, roll)` with `roll` in `[0, 1)`), or *generated now* inside the call from the default or a secure generator when no source is given. Either way, the same input can give a different result on each call unless the same randomness is supplied, and the name says so up front.<br><br>**Use when:** Load balancing across equivalent servers, picking a tip, quote, greeting, colour or avatar, choosing test or demo data, sampling a subset of records, A/B or experiment assignment, quiz questions and game draws, and retrying against a random replica. Put the thing being chosen after the verb (`takeRandomQuestion`) and the variant at the end: `takeRandomMany(items, n)` returns *n* distinct items (without replacement), `takeRandomWeighted(items, weights)` uses non-uniform odds, and `takeRandomSecure(items)` uses a cryptographically secure generator. Without a suffix, every candidate is equally likely. Accept the random source as an optional last parameter, so tests, replays and simulations can pass a seeded generator and get a repeatable result; document whether the default source is secure when the choice matters for fairness or security (prize draws, invitation codes, shard assignment that users could game). Leave the source collection unchanged by default and say so; when the chosen item must also leave the collection, such as drawing a card from a deck, follow with **remove** or say it in the name (`takeRandomAndRemove`). An empty collection is a caller error: **throw**, or offer `tryTakeRandom` (see **try**) that returns an empty result. Use **filter** or **find** to select by a condition, **slice** to select by position, **compute** or **find** for a *deterministic* best choice (`findBestOverload`, not `takeRandomOverload`), **sort** (`sortRandomly(items, rng)`) to put a whole collection in random order, and **create** to generate a brand-new random value that is not chosen from candidates (`createId`, `createSessionToken`, `createRandomInt(min, max)`).<br><br>**Why this word:** Randomness is the one fact the caller cannot see from the arguments: it makes the result change between calls, breaks snapshot tests, and decides whether a seed must be threaded through. Putting `Random` *in the verb*, the same way **try** puts the failure mode up front, makes that visible at every call site. `take` says "one of these existing items", which keeps it apart from generating a new random value, and `Random` says how it is chosen. A single word is not enough: `pick`, `choose` and `select` describe a deterministic choice just as well, and `random…` on its own (`randomItem`, `randomInt`) mixes picking from candidates with making a new value. `sample` is a statistics term that suggests several items or a distribution. Library functions (Python `random.choice`/`random.sample`, lodash `_.sample`, Rust `SliceRandom::choose`, Go `rand.Intn`) keep their names. | `pickRandom`, `chooseRandom`, `selectRandom`, `getRandom`, `randomChoice`, `randomElement`, `randomItem`, `sample` |
| **filter** | **Means:** Return the *subset* of a collection whose elements meet a condition, keeping their order. The input is not modified (`filterActiveUsers`, `filterByDate`).<br><br>**Use when:** Keeping or dropping elements by a predicate, including narrowing a set of options to the ones that apply and skipping irrelevant elements. Use **find** for the first match only, **count** when only the number of matches is needed, **takeRandom** to choose items by chance rather than by a condition, **remove** to modify the collection itself, and **slice** for a positional sub-range.<br><br>**Why this word:** `filter` is the universal functional name (JavaScript, Python, Java streams, Kotlin, Rust iterators). `select`, `narrow`, `skip`, `where` and `exclude` all mean "keep only what matters" and fragment the vocabulary. Query-language terms (SQL `WHERE`, LINQ `Where`) keep their names. | `select`, `narrow`, `skip`, `where`, `pick`, `keep`, `exclude`, `reject`, `omit`, `prune` |
| **count** | **Means:** Return *how many specific items* in a set, collection, sequence, tree or text meet a condition or belong to a kind, as a non-negative integer (`countActiveUsers(users)`, `countErrors(diagnostics)`, `countWhere(items, isOverdue)`, `countOccurrences(text, "TODO")`). The items being counted are always named: by the noun after the verb (`countUnreadMessages`), by a predicate argument (`countWhere`), or by a value to look for (`countOccurrences`). It only looks: the collection is not modified, no subset is built and returned, and calling it twice on the same data gives the same number. The grouped form returns one number per group instead of one overall (`countByStatus(orders)` → `{ open: 3, shipped: 7 }`); it still counts items and nothing else.<br><br>**Use when:** Badges and summaries ("3 unread"), limits and quotas (`countOpenConnections() < max`), statistics about matching elements, counting lines, words or occurrences in text, counting nodes of a kind in a tree, and tests that check how many results came back. Write `countX(items)` instead of `filterX(items).length`: it states the intent and does not allocate a throw-away list. Expect it to cost time in proportion to the data, because every item has to be examined, unless the type keeps the number up to date; if the count is reused, cache it or keep it as a field read with **get**. The *total* number of elements in a collection is not a count of specific items: it keeps the standard-library name (`length`, `size`, `len()`, .NET `Count`). Use **has** (`hasAny…`) when the question is only whether there is at least one, because it can stop at the first match; **filter** when the matching items themselves are needed; **find** for the first match; and **compute** when the items' *values* are combined (a sum of amounts, an average, a total price) rather than the items being counted. A number that is stored elsewhere and only retrieved is not counted by this code: use **fetch** for a count returned by a remote API (`fetchOrderCount`) and **read** for one read from a file or database (`readRowCount`).<br><br>**Why this word:** "Count" says exactly one thing: a whole number of items, found by looking at each one. `num…` and `numberOf…` read as nouns, so `numUsers` looks like a field rather than work that scans the data, and `tally` is the same act in a rarer word. Keeping counting out of **compute** makes the most common aggregation easy to find, and keeping it apart from `length`/`size` tells the reader that some items are left out by a condition, not the whole collection measured. Built-ins and query terms (`Array.prototype.length`, Python `list.count`, SQL `COUNT(*)`, LINQ `Count()`, lodash `countBy`) keep their names. | `num`, `numberOf`, `tally`, `howMany` |
| **fetch** | **Means:** Retrieve data *from a remote endpoint over HTTP* or a similar request/response network protocol (HTTPS, gRPC, GraphQL over HTTP, WebDAV, S3 and other object-store APIs, FTP): build a request, send it, wait for the response, and return its body, usually decoded (`fetchUser(id)`, `fetchOrders(query)`, `fetchReleaseNotes(version)`). The call crosses the network, so it is slow compared with local work, usually asynchronous, can time out, can be rate-limited, and can fail for reasons that have nothing to do with the caller's input (DNS, TLS, connection resets, 5xx responses). It *reads* remote state and does not change it: a `fetch…` function sends safe requests only (`GET`, `HEAD`, or a read-only query or RPC), so it can be retried or cached without side effects on the server.<br><br>**Use when:** REST and JSON API clients, GraphQL queries, gRPC unary reads, downloading files, images or packages, polling a status endpoint, reading from a CDN or object store, and loading remote configuration or feature flags. Name the function after the resource it returns, not the transport: `fetchUser`, not `fetchUserJson` or `httpGetUser`. Decode and validate the response inside the function or pass it straight to **parse**; callers should receive data, not a raw response object, unless returning the response is the point (`fetchRaw`). Make the failure mode visible: throw on network or non-success status (see **throw**), or name a variant that returns an empty result `tryFetch…` (see **try**). Accept a cancellation token, signal or timeout when the language supports it. Use **read** for local I/O (files, streams, pipes, an embedded or local database driver), **get** for data already in memory, and put a cache with **get** in front of a **fetch** when the result is reused. Requests that *change* remote state (`POST`, `PUT`, `PATCH`, `DELETE`, mutations) are not fetches: name them after their effect with **create**, **write**, **patch** or **remove** (`createOrder`, `removeComment`), and see *Messaging* under *Not covered* for fire-and-forget sends. Use **watch** for streaming or push-based connections (WebSocket, server-sent events, long polling) that deliver changes over time.<br><br>**Why this word:** The network is the most expensive and least reliable thing a function can touch, and callers need to know it is there before they call it in a loop, on a hot path or without error handling. `fetch` is the web-standard name for an HTTP request (the Fetch API, `fetch()` in browsers, Node, Deno and Bun) and is widely used for remote data loading, so readers already link it to the network. Using it *only* for network retrieval, and never as a general synonym of **get**, keeps that signal reliable: **get** is cheap and in memory, **read** is local I/O, **fetch** is a remote round trip. `download`, `pull`, `http`, `xhr` and `ajax` either name the same act or name the transport instead of the result. The built-in `fetch()` function, HTTP client methods (`axios.get`, `http.Get`, `requests.get`) and generated API-client names keep their names. | `download`, `pull`, `http`, `xhr`, `ajax` |
| **watch** | **Means:** Observe a file, directory, value or data source *over time* and get notified (callback, event, stream or channel) whenever it changes (`watchFile`, `watchConfig`, `watchQuery`).<br><br>**Use when:** File-system watchers, configuration reloaders, live queries, and reactive state subscriptions. Stop a watch with **unwatch**, or **close** the watcher object when the watch returned one. Use **handle** for the code that reacts to each notification.<br><br>**Why this word:** `watch` is the familiar term from build tools, file systems and reactive frameworks. `observe`, `monitor`, `subscribe` and `listen` would split one feature across several names. Library-required forms (`addEventListener`, RxJS `subscribe`) keep their names. | `observe`, `monitor`, `subscribe`, `listen`, `track` |
| **unwatch** | **Means:** The exact *opposite of* **watch**: stop observing something that an earlier `watch…` call started observing, so its callback, event, stream or channel receives no further change notifications (`unwatchFile`, `unwatchConfig`, `unwatchQuery`). It undoes one registration and nothing more. The file, value or data source is left untouched, the watching service (file-system watcher, config reloader, store, query engine) stays alive and keeps serving its other watches, and any notification already being delivered may still finish. After it returns, the program is in the same state as if that `watch…` call had never been made.<br><br>**Use when:** Ending a single watch that was started with **watch**, by naming the same target, and if needed the same callback, that was passed to it: `unwatchFile(path, onChange)` after `watchFile(path, onChange)`, `unwatchDirectory(dir)`, `store.unwatch(key, listener)`, `unwatchAll()` to drop every watch held by one owner. Name it after the same noun as its **watch** so the pair is easy to find: every `watchX` that is not torn down by closing a returned object should have an `unwatchX`. Typical callers are components being unmounted, projects being unloaded, files removed from a build, and configuration sections that are no longer needed. Make it idempotent: unwatching something that is not, or no longer, watched does nothing instead of throwing, so cleanup code can call it without checking first. If the watch returned its own handle (a watcher object, subscription or disposable), ending it through that handle uses **close** (`watcher.close()`), because the handle is a resource whose life ends there; **unwatch** is for asking the service that owns the watch to forget it. Use **close** to shut down the watching service itself, **finish** to end an operation that had a **start**, **remove** to take an item out of a collection, **clear** to empty a collection that stays usable, and **handle** for the code that reacts to each notification.<br><br>**Why this word:** A watch is a long-lived registration that leaks memory, file descriptors and CPU time if it is never undone, so its end needs a name that is as easy to find as its start. `un` + the same verb makes the pair self-evident: a reader who sees `watchConfig` can guess `unwatchConfig` without looking it up, and a search for `watch` finds both halves. `unsubscribe`, `unobserve`, `unlisten`, `unmonitor` and `untrack` are the reverses of **watch**'s own synonyms and would split the teardown across as many names as the setup was split before. `stop…`, `remove…` and `close…` already mean other things in this guide (ending an operation, deleting an item, releasing a resource). Library-required forms (`removeEventListener`, RxJS `unsubscribe`, `IntersectionObserver.unobserve`, Node `fs.unwatchFile`) keep their names. | `unsubscribe`, `unobserve`, `unlisten`, `unmonitor`, `untrack` |
| **can** | **Means:** A boolean predicate about *capability or permission*: is an operation possible, legal or supported for this subject (`canEdit(user, document)`, `canRetry`, `canConnect`)?<br><br>**Use when:** Authorisation checks, validation of allowed state transitions, feature support, and whether an action is allowed. Use **should** when the operation is possible and the question is whether to do it.<br><br>**Why this word:** Questions about what is possible are different from questions about what to do (see **should**) and from facts about the current state (see **is**/**has**). A dedicated prefix keeps those three readings apart. | `may`, `able`, `allows`, `supports` |
| **contains** | **Means:** A boolean predicate that says whether a collection, range, string, path or tree includes a given element, substring, value or descendant (`list.contains(item)`, `range.contains(date)`, `containsKey`).<br><br>**Use when:** Membership and inclusion tests where the argument is the thing searched for. Use **has** for the subject's own properties, and **find** when you need the matching element back rather than a yes/no.<br><br>**Why this word:** "Contains" says plainly which side is the container and which is the item, and it reads the same in every language. Using it for every membership test removes the `includes`/`in`/`within` split. Built-in APIs such as JavaScript's `Array.prototype.includes` keep their names. | `includes`, `in`, `within`, `inside` |
| **should** | **Means:** A boolean predicate that answers a *decision*: ought the program take some action now, given the current context (`shouldRetry`, `shouldCache`, `shouldShowBanner`)?<br><br>**Use when:** Policy, heuristic and feature-flag choices: whether to retry, cache, notify, render or skip. Use **can** when the question is whether something is possible or allowed at all, and **is** or **has** for plain facts.<br><br>**Why this word:** "Can" (possible) and "should" (advisable) often disagree. A separate prefix makes the policy points of the code easy to find and to change without touching the facts they are built from. | `needs`, `must`, `ought`, `wants`, `shall` |
| **is** | **Means:** A boolean predicate that says whether the subject *is* something: belongs to a kind, is in a state, or has an identity (`isEmpty`, `isAdmin`, `isExpired`). It answers a yes/no question and changes nothing.<br><br>**Use when:** State, kind and type checks on a single subject, including type-guard functions. Use **has** when the question is about owning a part or property, **can** for capability or permission, **should** for a decision about an action, and **contains** for membership in a collection.<br><br>**Why this word:** `is` is the most direct yes/no prefix in every language. Keeping plural or past-tense variants (`are`, `was`) out means every predicate starts with one of five known prefixes and can be found by searching for them. | `are`, `was`, `were`, `does` |
| **has** | **Means:** A boolean predicate that says whether the subject *owns* a part, property, member, flag or child (`hasPermission`, `hasChildren`, `hasDiscount`), or whether at least one or all of its parts meet a condition.<br><br>**Use when:** Asking about something the subject carries or is made of. For "at least one element satisfies P" use `hasAny…` (`hasAnyErrors`) instead of `some…`; for "every element satisfies P" use `hasAll…` (`hasAllPermissions`) instead of `every…`/`areAll…`. Use **count** when the question is *how many* parts meet the condition, **contains** when the question is whether a collection includes a given element, and **is** when it is about what the subject itself is.<br><br>**Why this word:** Possession is a different question from identity. Keeping `has` for it lets `isX` versus `hasX` carry real meaning (`isLocked` versus `hasLock`), and it replaces the vaguer `some…`/`every…` with a readable `hasAny…`/`hasAll…` pair. | `some`, `every`, `all`, `areAll`, `owns`, `holds`, `possesses` |
| **values** | **Means:** Return the *values* of a map or the elements of a collection, without their keys, as an iterator or list (`values`).<br><br>**Use when:** Whenever a collection type exposes its stored values. Name the method exactly `values`, never `getValues`/`listValues`/`toArray`.<br><br>**Why this word:** `values` is the standard-library name in many languages (JavaScript `Map.prototype.values`, `Object.values`, Python `dict.values()`, Java `Map.values()`). Matching it exactly lets every collection type be used the same way. | `getValues`, `listValues`, `elements`, `toArray` |
| **initialize** | **Means:** Prepare an *existing* object, module, service or subsystem so it is ready for use: fill in its starting state, load defaults, open required dependencies, compute one-time data. It may be done lazily the first time it is needed (`initializeApp`, `initializeDatabase`).<br><br>**Use when:** Application and module startup, one-time setup routines, and lazy setup on first use (`initializeCache` instead of `ensureCache`). Use **create** to allocate something new and **start** to set something running. Language-required forms (Go `init()`, Python `__init__`) keep their names.<br><br>**Why this word:** The full word reads clearly, while `init`, `setup` and `ensure` are abbreviations or vaguer. `ensure` in particular hides whether the call creates something or just checks. | `init`, `ensure`, `setup`, `prepare`, `bootstrap` |
| **finish** | **Means:** Bring an *in-progress operation* to its end and settle its final state, whether it completed normally or is being ended early (`finishUpload`, `finishRequest`, `finishSession`).<br><br>**Use when:** Ending requests, uploads, sessions, transactions, phases and other operations that had a matching **start**. When ending early is the point, say so in the rest of the name or a parameter (`finishUploadCancelled`, `finish({ cancelled: true })`). Use **close** to release a resource.<br><br>**Why this word:** "Finish" names the matching end of **start** and says the operation is over. `end`, `stop`, `exit` and `cancel` differ only in *why* it ended, which belongs in the rest of the name. | `end`, `stop`, `cancel`, `exit`, `complete`, `done`, `terminate`, `halt`, `abort` |
| **start** | **Means:** Set something *running* that keeps going after the call returns: a server, worker, timer, watcher, background job, session or connection (`startServer`, `startTimer`, `startSession`).<br><br>**Use when:** Beginning long-running or asynchronous work and opening sessions or connections. Its counterpart is **finish** for operations and **close** for resources. Use **run** when the call blocks until the work is done.<br><br>**Why this word:** `begin` and `open` both mean "make it active", and having three words for it makes the matching end harder to guess. With `start`, the pairs are `start`/`finish` and `start`/`close`. Standard-library forms (`open()` for files, `BEGIN` in SQL) keep their names. | `begin`, `open`, `launch`, `activate`, `enter` |
| **next** | **Means:** Return the element *after the current one* in an ordered sequence, such as an iterator, cursor, stream, token list, page set or ID series. A stateful object also advances its position. When the sequence is exhausted it returns an end marker (`done: true`, `null`/`None`/`nil`, `false`, or a `(value, ok)` pair) or throws, depending on the language's iterator convention (`iterator.next()`, `nextToken()`, `nextPage(cursor)`, `nextId()`).<br><br>**Use when:** Iterators and generators, lexers and parsers that consume input one token at a time, pagination (`nextPage`), sequence and ID generators (`nextId`), and stepping through states or scheduled occurrences (`nextRetryDelay`, `nextOccurrence(date)`). Conditional or look-ahead variants go in the rest of the name (`nextIf(predicate)`, `nextTokenIsComma`). Use **find** to search ahead for an element that meets a condition, **get** to read the current element without advancing, and **compute** when the following value has to be calculated rather than read from a sequence.<br><br>**Why this word:** `next` is the established iterator vocabulary across languages (JavaScript/TypeScript, Python `__next__`, Java `Iterator.next()`, Rust `Iterator::next`, Go `iter.Pull`'s `next`), so readers recognise it immediately as "move forward one step". `advance`, `step` and `successor` mean the same and would split one idea across several names. Language-required forms (C# `IEnumerator.MoveNext`, Python `__next__`) keep their names. | `advance`, `step`, `forward`, `successor`, `succ`, `following` |
| **format** | **Means:** Turn a value into *human-readable text*, without writing it anywhere, and return the string (`formatDate`, `formatCurrency`, `formatDuration`).<br><br>**Use when:** Display strings, user-facing messages, pretty-printed output and debug descriptions. Then use **write** or **log** to output the string. Use **serialize** when the text must be machine-readable and readable back with **parse**. Language-required forms (Go `String()`, Python `__str__`/`__repr__`) keep their names.<br><br>**Why this word:** "Format" says the output is meant for people and its exact shape may change, a promise callers need to know about. `stringify` suggests a JSON-style round trip, and `string`/`show` are vague. | `string`, `pretty`, `display`, `show`, `describe`, `repr` |
| **close** | **Means:** Release the resources an object holds, such as file handles, connections, sockets, watchers, processes, locks or subscriptions, and end its life. After `close` the object must not be used again (`closeConnection`, `file.close()`).<br><br>**Use when:** Cleanup and teardown of anything that owns a resource. Pair it with **start** (or with **create**/**read** for opened resources). Use **finish** to complete an operation, and **clear** to empty something that stays usable.<br><br>**Why this word:** `close` reads naturally for files, connections and streams, and it is the standard term in most I/O libraries. `dispose`, `release` and `free` all mean "done with it, give it back". Language-required forms (C# `Dispose`, Python `__exit__`, Rust `Drop`) keep their names. | `dispose`, `release`, `free`, `destroy`, `shutdown`, `teardown`, `cleanup` |
| **run** | **Means:** Execute a task, command, program, script or process *to completion*, usually blocking until it finishes and returning its result (`runMigrations`, `runCommand`, `runJob`).<br><br>**Use when:** Entry points, command execution, running a child process or a batch of work: `runProcess` rather than `spawnProcess`, `runBuild` rather than `doBuild`. Use **start** when the call returns while the work keeps going in the background.<br><br>**Why this word:** `do…` is empty and `spawn`/`exec` are platform terms. `run` tells the reader that the work happens now and is over when the call returns. | `spawn`, `do`, `execute`, `exec`, `perform`, `invoke` |
| **read** | **Means:** Bring data *in from outside the process*: from a file, stream, socket, pipe, database, environment or remote service. This involves I/O and may block or fail (`readFile`, `readConfig`, `readLine`). Retrieving data with a request/response call over HTTP or a similar protocol is **fetch**.<br><br>**Use when:** File and stream reads, loading configuration or resources from disk, reading rows from a database, importing data files. Use **get** when the data is already in memory and **parse** to interpret the bytes once they have been read.<br><br>**Why this word:** Separating I/O (**read**) from in-memory access (**get**) tells callers where latency and errors can come from. `load` and `import` add no meaning beyond "read and keep". `read` is the standard-library term almost everywhere. | `load`, `import`, `ingest`, `slurp` |
| **write** | **Means:** Send text or data *out* to a destination such as a file, stream, buffer, console, socket or response body (`writeFile`, `writeHeader`, `writeReport`). The side effect is the point; the return value is at most a count or an error.<br><br>**Use when:** Output of any kind: saving files, printing to the console, streaming a response, generating a report into a writer. The verb is the same whatever the destination: `writeHelp` rather than `printHelp`, `writeFile` rather than `saveFile`. Use **format** to build a string without outputting it, and **log** for diagnostic logging.<br><br>**Why this word:** `print`, `save` and `output` differ only in destination, which belongs in the noun or parameter. `write` is the stream and writer vocabulary of nearly every language's standard library. | `print`, `output`, `save`, `persist`, `store` |
| **try** | **Means:** A *prefix* that marks a fallible variant of another verb. Instead of throwing, panicking or reporting, it returns an empty result (`null`, `None`, `nil`, `false`, an `Option`/`Result`, or Go's `(value, ok)`) when the operation can't be done (`tryParseInt`, `tryGetUser`, `tryConnect`).<br><br>**Use when:** Always together with the real verb: `tryGet…`, `tryParse…`, `tryRead…`. It is the non-throwing counterpart of **throw**. Use it when "not found or not applicable" is a normal outcome the caller is expected to handle.<br><br>**Why this word:** Putting the failure mode at the *front* of the name makes it visible at every call site. Suffixes such as `…OrNull`/`…OrNil` are easy to miss and grow inconsistent. | `maybe`, `attempt`, `safe`, `orNil`, `orNull` |
| **find** | **Means:** Search a collection, tree, file system, text or data store for the *first* item or position that meets a condition, and return it, or `null`/`None`/`nil`/`-1` when nothing matches (`findUserByEmail`, `findConfigFile`).<br><br>**Use when:** Lookups that scan or query. Positional variants go in the name (`findIndex`, `findLast`, `findLastIndex`), not separate verbs like `indexOf`. Use **filter** to get *all* matches, **contains** for a yes/no, and **get** for direct keyed access.<br><br>**Why this word:** "Find" tells the reader the call is a search that may come up empty and may cost time in proportion to the data, unlike **get**. Built-ins such as `indexOf` or SQL `SELECT` keep their names. | `search`, `lookup`, `locate`, `index`, `indexOf`, `last`, `lastIndexOf`, `seek`, `query` |
| **clone** | **Means:** Make an *independent copy* of an existing object, so changes to the copy do not affect the original (`user.clone()`, `cloneSettings`). It may change some fields on the copy. Each type should document whether the copy is deep or shallow.<br><br>**Use when:** Duplicating records, configs or state before modifying them, including immutable "with" helpers: write `cloneWithEmail` rather than `withEmail`. Use **create** for objects that are not based on an existing one.<br><br>**Why this word:** "Clone" says "same shape, separate identity" unambiguously. `copy` can also mean copying bytes into an existing buffer, and `with…` hides that an allocation happens. Language-required forms (Java `clone()`, Python `__copy__`) keep their names. | `copy`, `with`, `duplicate`, `dup`, `replicate` |
| **patch** | **Means:** Apply a set of *changes to something that already exists*: change the given fields or parts and leave everything else as it was. The change is done either in place or by returning the patched version (`patchUser(id, { email })`, `patchStatus(order, "shipped")`, `patchSettings(changes)`). This covers changing a single field (a setter), several fields, a child element, or swapping in fresh data from the source.<br><br>**Use when:** Setters, partial updates, applying a diff or a list of edits, replacing a child element, and refreshing stale data: `patchEmail(user, email)` or `patch(user, { email })` rather than `setEmail`/`updateUser`. Pass the changes as an argument, so the call site shows exactly what is being changed. Say in the name or documentation whether it mutates in place or returns a new version. Use **add** when the item is new to the collection, **clear** to empty it, **initialize** for the first setup, and **clone** for a modified copy that leaves the original alone.<br><br>**Why this word:** "Patch" says two things: the target already exists, and only the listed parts change. That is exactly what a reader needs when looking for side effects, and it matches HTTP `PATCH` and diff/patch tooling. `update`, `set`, `replace`, `apply`, `refresh` and `change` all name the same "make this existing thing different" act, but none of them says how much of it changes. Accessors a framework requires (JavaBean `setX`, React `setState`, an ORM's `update()`) keep their names. | `update`, `set`, `replace`, `apply`, `refresh`, `change`, `modify`, `mutate`, `edit`, `alter`, `assign` |
| **to** | **Means:** Convert a value into a *different type or representation* and return the result, leaving the source unchanged (`toString`, `toJson`, `toDto`). The word after `to` names the target.<br><br>**Use when:** Every conversion, written so the target comes last: as a method `value.toTarget()`, or as a free function `sourceToTarget(value)` (`celsiusToFahrenheit`, `userToDto`). A conversion *from* something is written the same way with the source first (`dtoToUser`, not `fromDto`). Use **format** when the target is human-readable text and **serialize** when it is a storage or wire format.<br><br>**Why this word:** Reading left to right, `sourceToTarget` states the direction, which `convert`, `transform` and `map` all leave out. It also agrees with the standard-library `toString` found in many languages. | `as`, `from`, `transform`, `map`, `convert`, `normalize`, `cast`, `coerce`, `translate` |
| **check** | **Means:** Examine something for problems and *act on what it finds as a side effect*: report, record, throw or fail loudly (`checkHealth`, `checkInvariants`, `checkPermissions`). It usually returns nothing, or a value learned along the way, not a pass/fail result for the caller to interpret.<br><br>**Use when:** Linting and analysis passes, health checks, invariant and precondition guards that throw, and assertions in tests. Use **validate** to test *input data* against rules and return the result, **throw** (`throwIfInvalid`) for a helper whose only job is to raise the error, and **is**/**has** for side-effect-free predicates.<br><br>**Why this word:** "Check" suits examination that reports or enforces rather than returns. Folding `assert`, `inspect` and `examine` into it leaves exactly two correctness verbs, each with a clear contract. Test-framework APIs (`assert.equal`, `expect`) keep their names. | `assert`, `inspect`, `examine`, `analyze` |
| **parse** | **Means:** Read structured data from raw text or bytes and build the in-memory model it describes (`parseConfig`, `parseDate`, `parseArgs`). It may fail on malformed input and must report or return that error.<br><br>**Use when:** Turning JSON, YAML, CSV, query strings, command-line arguments, dates, numbers, source code or binary formats into typed objects. It is the inverse of **serialize**. Use **read** for the I/O that fetches the bytes, then **parse** to interpret them.<br><br>**Why this word:** "Parse" is the widely understood term and is clear about direction. `unmarshal`, `decode` and `deserialize` add library- or language-specific flavour without changing the meaning. Methods required by an interface (Go `UnmarshalJSON`) keep their names. | `unmarshal`, `decode`, `deserialize`, `unpack` |
| **handle** | **Means:** React to an incoming *event, request, message or callback* and do whatever work it calls for (`handleClick`, `handleRequest`, `handlePaymentSucceeded`).<br><br>**Use when:** UI event listeners, HTTP/RPC route handlers, queue and message consumers, webhook receivers and callback bodies. Use **run** for work you start yourself rather than in response to an outside trigger.<br><br>**Why this word:** `on…` names when something fires, `provide…` names what comes back, and `process…` says almost nothing. `handle…` names the responsibility, which is what the reader needs. Names a framework requires (`onClick` props, `provideX` interface methods) keep their names. | `on`, `provide`, `process`, `respond`, `serve`, `dispatch`, `react` |
| **remove** | **Means:** Take one or more specific items *out* of a collection or structure, which keeps existing without them (`removeItem(cart, item)`, `removeMember(team, user)`).<br><br>**Use when:** Removing map keys, list elements, records from a set, children from a tree, or the top of a stack (`removeLast`). Use **clear** to remove *everything*, **close** to release a resource, and **filter** to get a copy with some items left out.<br><br>**Why this word:** It is the exact inverse of **add**, so the pair `addX`/`removeX` is predictable. `delete` and `pop` describe the same act, so they are folded in. Built-ins and protocol terms such as `Map.prototype.delete` or HTTP `DELETE` keep their names. | `delete`, `pop`, `erase`, `drop`, `discard`, `evict`, `detach` |
| **add** | **Means:** Put a new item into an *existing* collection or structure, growing it (`addItem(cart, item)`, `addMember(team, user)`).<br><br>**Use when:** Adding to lists, sets, maps, stacks, queues and trees, at any position: `addLast`, `addFirst`, `addAt(index, item)` or `addSorted` instead of append/insert/push. Use **patch** to change an item that is already there, and **create** for a fresh collection.<br><br>**Why this word:** Where an item goes is a detail of the data structure, not a different action. Spelling it as a suffix on one verb keeps the paired name obvious (`add`/`remove`) and the collection API small. Built-ins such as `Array.prototype.push` or Python's `list.append` keep their names. | `append`, `insert`, `push`, `fill`, `put`, `attach`, `enqueue`, `prepend` |
| **clear** | **Means:** Remove *all* contents or state from an object while the object itself stays usable, returning it to its empty or initial condition (`clearCache`, `clearForm`, `clearSelection`).<br><br>**Use when:** Emptying caches, maps, buffers, forms and pending queues, and resetting counters or flags to their starting values. Use **remove** for specific items, **close** when the object is being thrown away, and **initialize** for the first setup.<br><br>**Why this word:** `clear` (the standard collection method in most languages) and `reset` describe the same end state: empty and ready for reuse. One verb avoids guessing which word a given type chose. | `reset`, `empty`, `wipe`, `purge`, `truncate` |
| **create** | **Means:** Build and return a *new* object, record, value or resource that did not exist before (`createUser`, `createConnection`, `createInvoice`). The caller owns the result.<br><br>**Use when:** Factory functions and constructor helpers, including making an instance from a template or generic type. Use **clone** when the new object is a copy of an existing one, **initialize** to prepare something that already exists, and **parse** when the object is built from text or bytes. Language keywords and required forms (`constructor`, `__init__`, `new`) are not affected.<br><br>**Why this word:** `new…`, `make…`, `build…` and `create…` all name the same act. One verb makes allocation points easy to search for, and it reads naturally in any language. | `new`, `make`, `build`, `instantiate`, `construct`, `produce`, `alloc` |
| **get** | **Means:** Return a value that already exists and is cheap to reach: a field, a cached result, or an entry in an in-memory map or list. Calling it has no observable side effects and calling it twice returns the same thing.<br><br>**Use when:** Accessors and in-memory lookups: `getUser(id)` from a loaded map, `getSetting(key)`, `order.getStatus()`. If the value has to be calculated, use **compute**; if it comes from disk or a database, use **read**; if it comes over HTTP or a similar network protocol, use **fetch**; if "not found" is a normal outcome, use **try** (`tryGetUser`). Languages that prefer bare-noun accessors for plain fields (Go `user.Name()`, Python/C#/Kotlin properties) should follow that idiom, and keep `get` for lookups that take arguments or do real work.<br><br>**Why this word:** `get` is the most common verb in real codebases, so readers already expect it to mean a cheap, side-effect-free read. One word for this keeps the cheap path recognisable and makes anything named differently stand out as more expensive. | `retrieve`, `obtain`, `acquire`, `access`, `grab` |
| **compute** | **Means:** Work out a *new result from other data* through calculation or reasoning. The result was not stored anywhere and has to be derived (`computeRoute`, `computeDiff`, `computeLayout`). This explicitly includes *reducing many values into one*: totals, averages, summaries, and merging several maps, configs or lists into a single result (`computeTotal(items)`, `computeMergedConfig(layers)`, `computeSummary(events)`). It also covers resolving references and inferring values.<br><br>**Use when:** Any derived value worth naming as work: arithmetic and statistics, diffs, layouts, resolved dependencies, and every many-to-one aggregation (sum, fold, merge, collect into one structure). Implement it however suits the language, such as a loop or a built-in `reduce`/`fold`/`sum`, but name the function `compute…` after the result it produces. Use **get** for values that are already available, **hash** for a fixed-size digest of data, **count** for how many items of a kind a collection holds, **filter** to keep a subset without combining it, **to** for a one-to-one conversion, and put a cache with **get** in front of a **compute** when the result is reused.<br><br>**Why this word:** `compute` tells callers that a new value is being derived and that the call costs something, so the result may be worth caching. Calculating, resolving, inferring, reducing, merging and aggregating are all that same derive-a-result act. Naming the function after its result (`computeTotal`) says more than naming the technique (`reduceItems`). Built-in methods such as `Array.prototype.reduce`, Kotlin `fold` or Python `sum` keep their names. | `resolve`, `infer`, `calculate`, `calc`, `derive`, `determine`, `evaluate`, `reduce`, `fold`, `aggregate`, `accumulate`, `collect`, `combine`, `merge`, `join`, `sum` |

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
    with `Promise.all(map(items, fetchItem))`; sequence dependent work by folding
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
      (candidate) => Math.abs(candidate.column - computeMiddleColumn(grid)),
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
   are tool directives, such as `// eslint-disable-next-line`, the note on each
   exception in section 15, and the public JSDoc of rule 6.
6. **Keep public JSDoc when refactoring.** When you refactor a codebase that
   documents its public API with JSDoc — a `/** … */` block on an exported
   function, type or constant — NEVER remove those comments. Callers read them
   in their editors and documentation tools. Keep each one on the export it
   documents. If the refactor changes the signature, for example by injecting
   an effect as a new argument (section 7), update the comment to match.

## 12. Tests

1. **One runner, imported by name.** With Vitest, `import { describe, expect, it } from 'vitest';`.
   NEVER use mocks, spies, `beforeEach`, `afterEach`, snapshots, `.only`,
   `.skip` or `it.each`. A body that takes its effects as arguments (section 7)
   needs no mock: pass a stub function.
2. **One `describe` per file, named exactly after the function:**
   `describe('getTopCandidate', …)`. Only a spec that checks behaviour across
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
   - for an entry point or a `patch…` step: that the input it was given is
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
     const filterCandidates = (rows: string[], layout: Layout) =>
       filterGridCandidates({ grid: createGrid(rows), layout }).candidates;
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
anything the refactor does not touch. Keep every public JSDoc comment on the
export it documents (section 11.6).

1. **Name the steps.** Read the body top to bottom, find each distinct
   transformation and extract it into a small named `const` arrow, in its own
   file when the surrounding feature has a folder. Name each one with the verb
   the walk in section 8.6 gives. Then check every other function name in the
   file the same way, rename the ones that break section 8.6, and update every
   caller.
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
10. **Run the typecheck, the linter and the specs** for the package, and add the
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
- A function name that does not start with a verb from the verb table, that
  starts with a word from its **Synonyms** column, or whose verb is not the
  first one the walk in section 8.6 reaches.
- A comment that explains the code instead of a better name or a test. Public
  JSDoc is not such a comment and stays (section 11.6).

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

Fill in these four blanks, then the guide is complete for that repository:

1. **The dependencies.** `lodash-es`, `ts-pattern` and `ramda` in
   `dependencies`; `@types/lodash-es` in `devDependencies`, without which the
   named lodash imports do not type-check.
2. **The `chain` wrapper module** (section 10.11) and its import path.
3. **Where constants live** (section 2.8): the path of the nearest `consts.ts`
   per package.
4. **The real commands** for formatting, linting, typechecking and testing,
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
