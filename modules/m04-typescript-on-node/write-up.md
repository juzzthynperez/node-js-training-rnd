# M04 Write-Up — TypeScript Basics & TypeScript on Node.js

> Filled from the actual build in `node-js-training-ground` (PR #8). Environment: Node
> 24.21.0 through `fnm exec --using=24 -- …`, TypeScript 7.0.2, `@types/node` 24.19.2,
> tsx 4.23.15.

## What I built

A strict-TypeScript project under `modules/m04-typescript-on-node/` in the ground repo:

- `tsconfig.json` — the strict config the brief specifies (`strict`,
  `module`/`moduleResolution: nodenext`, `verbatimModuleSyntax`, `erasableSyntaxOnly`,
  `noUncheckedIndexedAccess`, `noEmit`, `skipLibCheck`) plus `target`/`lib: ES2023`,
  `types: ["node"]`, and `allowImportingTsExtensions` (needed so relative imports can
  carry the `.ts` extension — that's what both tsx and Node's native stripper require,
  and it's only legal because `noEmit` is on).
- `package.json` — `type: module`, `engines.node >= 24`, and three scripts:
  `typecheck` (`tsc --noEmit`), `toolkit` (tsx), `toolkit:native` (`node`).
- `async-toolkit.ts` — the M02 JavaScript toolkit ported with generics: `sleep(ms)`,
  `retry<T>`, `withTimeout<T>`, `mapLimit<T, R>`, the `TimeoutError` class, and the full
  node:assert self-check harness (typed).
- `task-domain.ts` — `Task` (an `interface`), `CreateTaskInput` and `UpdateTaskInput`
  derived from it with `Omit`/`Partial`, the discriminated-union `Result<T, E>` with
  `ok`/`err` constructors and `isOk`/`isErr` type guards, and five runtime self-checks.
- `demo-native-strip.ts` — a small consumer of the toolkit that exists to be run both
  ways: `npx tsx demo-native-strip.ts` and `node demo-native-strip.ts`.

All gates passed: zero `tsc --noEmit` errors, 16/16 toolkit self-checks via tsx, 5/5
domain self-checks, and — the awareness-level checkpoint — the native type-stripping run
completed all 16 checks on Node 24.21.0 with exit 0. Evidence lives in the trainer-private
`evidence/m04/` (INDEX.md lists every artifact).

## Why it's built this way (key decisions)

- **Utility types for the input types, not hand-written copies.**
  `CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt'>` and
  `UpdateTaskInput = Partial<Omit<Task, 'id' | 'createdAt' | 'updatedAt'>>`. Hand-writing
  them would have meant three copies of the same field list that drift the moment `Task`
  changes — the compiler now enforces the relationship instead of me remembering it.
  `Partial` layered *outside* `Omit` (not `Omit<Partial<Task>, …>`) matters: the system
  fields must be gone, not merely optional.
- **`Result<T, E>` as a discriminated union on a literal field.**
  `{ ok: true; value: T } | { ok: false; error: E }` — `ok` is a *literal* type
  (`true`/`false`), which is what makes `switch (r.ok)` narrow both arms with no casts.
  Modeling it as a class hierarchy or a single `{ value?: T; error?: E }` was the
  alternative; the optional-bag version type-checks invalid states (both fields set, or
  neither), which defeats the point. What would change if modeled differently: an
  optional-bag `Result` needs runtime assertions at every use, and a class version can't
  narrow in a `switch` without `instanceof` chains.
- **`interface Task` but `type` for everything derived.** The object-shaped,
  potentially-extensible domain model is where `interface` fits (declaration merging is
  possible if a later module needs it); unions, `Omit`/`Partial` applications, and the
  `Result` union *can only* be `type`. That's the honest version of "type vs interface"
  rather than a blanket rule.
- **`mapLimit` switched from index reads to a shared iterator.** Under
  `noUncheckedIndexedAccess`, `items[index]` is `T | undefined`, and the loop condition
  `index < items.length` does **not** narrow it — the compiler has no idea the guard and
  the access are related. Options were `items[index]!` (asserting the invariant nobody
  can see), a `!== undefined` guard (which would silently *skip* a legitimately-undefined
  element), or restructuring. Workers now pull from `items[Symbol.iterator]()`:
  `step.done === false` narrows `step.value` to `T` by definition, and since there's no
  `await` between the pull and `nextIndex++`, the input-order guarantee survives exactly
  as before (the order self-check passes).
- **`retry`'s `lastError` declared as `unknown` — with no seed, after validating the
  claim first.** My draft comment asserted that `let lastError: unknown;` before a
  `throw` would trip TS2454 (definite assignment can't prove the loop ran), and seeded
  `= undefined` to appease a check that I never actually saw fire. During validation I
  wrote a minimal repro under TS 7.0.2 `--strict`: the loop-then-use pattern with
  `unknown` compiles clean — **TS2454 exempts `unknown`** (its domain already includes
  `undefined`, so "possibly unassigned" isn't out of type), while the same pattern with
  `let lastError: string` *is* flagged. So the seed came out and the comment now states
  the verified behavior: no `any`, no `!`, no `= undefined`, and the loop always runs
  at least once (`retries >= 0` is checked) so a real error is always captured. The
  lesson I'd repeat: a compiler-behavior claim in a comment deserves a three-line
  repro, not confidence.
- **Native type stripping on Node 24 — what actually happened.**
  `node async-toolkit.ts` and `node demo-native-strip.ts` both ran to completion on
  Node 24.21.0: all 16 self-checks, exit 0, **no experimental warning printed**. That
  matches `erasableSyntaxOnly` doing its job — the port contains no enums, no
  namespaces, no parameter properties, so there was nothing for the stripper to reject.
  This is still awareness-level for this cohort: the brief positions full stability at
  Node 26, and "it worked today on a file this small" is a weaker claim than "it's
  stable." The demo file uses an explicit `./async-toolkit.ts` extension in its import —
  native stripping resolves `.ts` imports by exact path, no extension guessing.

## How to build it (teach it to the next trainee)

### Deriving input types from a domain type

Start from the field list and ask, for each field: *who assigns this?* The caller
assigns `title` and `status`; the system assigns `id`, `createdAt`, `updatedAt`.

```ts
interface Task {
  readonly id: string;
  title: string;
  description?: string;
  status: TaskStatus;
  readonly createdAt: string;
  readonly updatedAt: string;
}

type CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateTaskInput = Partial<Omit<Task, 'id' | 'createdAt' | 'updatedAt'>>;
```

The reasoning, not just the syntax:

1. `Omit<Task, …>` produces a type that *stays connected* to `Task`. Add a
   `priority?: number` field to `Task` and `CreateTaskInput` gains it automatically —
   with a hand-written copy you'd have to remember, and one forgotten edit shows up as a
   runtime surprise instead of a compile error.
2. `Partial` goes on the *outside*: `Partial<Omit<Task, 'id' | …>>` makes the remaining
   fields optional while system fields stay nonexistent. The reverse
   (`Omit<Partial<Task>, …>` — same result here, but the layering habit pays off later)
   and `Partial<Omit<Task, 'id'>>` (id optional → callers *may* supply an id) are the
   two ways to get it subtly wrong.
3. `readonly` on the system fields survives the derivations — `Omit` and `Partial` both
   preserve it. Callers can't assign to `id` even on a patch.

Practice the same shape with your own domain: pick a `Post` or `Invoice`, derive
`CreatePostInput`/`UpdatePostInput`, and check by deliberately adding a field to the
parent — if the derived types don't pick it up, your derivation is wrong.

## Concepts worth explaining

**`unknown` vs `any` vs `never`.** `any` is *opting out of the type system* — it
assigns to and from everything, and one `any` spreads: the compiler stops checking the
surrounding code. `unknown` is the honest version of the same flexibility: it also
assigns *from* anything, but assigns *to* nothing — you must narrow it (`instanceof`,
`typeof`, a custom guard) before it's useful. That's why `catch (e)` gives you `unknown`
under `strict` and why the self-check harness does
`error instanceof Error ? error.message : String(error)`. `never` is the opposite
endpoint: the type of values that *don't exist* — the return of a function that always
throws, or the empty set left after a union is fully narrowed. In
`switch (outcome.ok)` with `Result<T, E>`, after handling `case true` and
`case false`, any further narrowing would be `never`.

**`type` vs `interface`.** Both describe object shapes and mostly overlap for plain
data. `interface` is declaration-merging: two `interface Task` blocks combine, which
matters for extending library types. `type` is an alias — it can be a union, a mapped
type, `Omit<…>` applied to something else, a tuple, a primitive. So the rule of thumb
used in `task-domain.ts`: object-shaped domain models get `interface`; everything the
compiler *computes* gets `type`. `Result<T, E>` must be `type` (it's a union);
`CreateTaskInput` must be `type` (it's an `Omit` application); `Task` can be either,
and `interface` is chosen deliberately to make the split visible.

## What tripped me up

- **The first typecheck failed with one error:**
  `async-toolkit.ts(270,20): error TS18046: 'error' is of type 'unknown'.` — in the
  `assert.rejects(…, (error) => { … })` validator I read `error.name` before narrowing.
  `assert.rejects`'s validator parameter is typed `unknown`, exactly like `catch`. The
  fix was reordering: `assert.ok(error instanceof Error)` first (assert.ok is an
  assertion signature, so it narrows), then read members. The compiler wasn't being
  pedantic — a non-Error rejection (a string, say) would genuinely have no `.name`.
  (A *planted* error — `TS2322: Type 'number' is not assignable to type 'string'` — was
  separately captured in `evidence/m04/sample-compiler-error.txt` as a clean example of
  the error format for teaching; the TS18046 above was the real first-run failure.)
- **I documented a compiler behavior that wasn't true — until validation caught it.**
  The original `lastError` comment claimed TS2454 would fire on the unassigned
  `unknown`; a minimal repro proved TS7 exempts `unknown` entirely (see key decisions).
  The false claim had survived a full build because nothing ever *tested* it — the seed
  masked the question. Both the comment and this entry now state the verified behavior.
- **`noUncheckedIndexedAccess` doesn't believe your loop guard.** `while (i < xs.length)`
  still gives `xs[i]: T | undefined`. Every instinct says add `!`; every `!` is a
  promise the compiler can't check. The iterator rework (see key decisions) was the
  discovery of this module — express the invariant in the *type*, not in a comment.
- **`tsx` vs native type stripping on Node 24.** tsx is a dev runner: it transforms
  types away through esbuild, accepts extensionless and `.ts` imports, works on any
  syntax, and its failure modes are runtime/module-resolution problems. Native type
  stripping is Node itself erasing annotations at load time: erasable syntax only
  (anything non-erasable is a hard `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`), explicit `.ts`
  import paths, and *no type-checking whatsoever* — it proves the file runs, not that
  the types are right. Both produced identical output for the demo; only `tsc --noEmit`
  actually checks.
- **The fnm default trap, third module running.** Fresh shell answered v22 while
  `.nvmrc` pins 24. Every command went through `fnm exec --using=24 -- …` after a
  `node --version` probe — and on this setup `fnm exec --using=24 -- npm …` fails to
  spawn ("program not found") because fnm's direct exec doesn't resolve the Windows
  npm shim; wrapping in a shell fixes it:
  `fnm exec --using=24 -- bash -c 'npm run typecheck'`.

## Checkpoint evidence

All artifacts in trainer-private `evidence/m04/` (INDEX.md indexes them):

1. **Zero `tsc --noEmit` errors** on the strict config — `typecheck-zero-errors.txt`
   (shows Node v24.21.0 and a clean `tsc --noEmit`, exit 0).
2. **Toolkit runs via tsx** — `toolkit-via-tsx.txt`: all self-check groups
   (`sleep`, `retry`, `withTimeout`, `mapLimit`) report passing; the summary line is
   your number, not a target — record whatever yours says.
3. **Native type-stripping attempt on Node 24** — `toolkit-native-stripping.txt`:
   `node async-toolkit.ts` completed the full run with exit 0 and no warning on
   v24.21.0. Plus the both-ways demo: `npx tsx demo-native-strip.ts` and
   `node demo-native-strip.ts` produced identical output
   (`retry → 7`, `mapLimit → [2, 4, 6]`, `withTimeout → TimeoutError 25ms`, `demo ok`).

## What I'd do differently

- **Try native stripping on a deliberately non-erasable file first.** Everything worked
  first try because `erasableSyntaxOnly` was on from the start — but that means I never
  *saw* the stripper's failure mode. A five-line file with a tiny `enum` (or a
  constructor parameter property) run under `node file.ts` would have produced the
  `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` error worth showing trainees, rather than only
  reading about it.
- **Type the options objects from the start instead of inline.**
  `retry`'s `{ retries?: number; delayMs?: number }` sits inline in the signature; an
  exported `RetryOptions` interface would give trainees a named thing to read and gives
  future callers IntelliSense on the options bag.
- **Wire `npm run domain` as a script.** The domain self-checks run through
  `npx tsx task-domain.ts` — a one-line script would make the second deliverable as
  discoverable as the toolkit.
- **Keep the order self-check as a named invariant.** The iterator rework was correct,
  but it silently *could* have broken input-order preservation; the passing check is the
  only thing proving it didn't. I'd add an order check with an obviously-unsorted input
  (e.g. items whose delays are inversely proportional to their value) so a regression
  can't pass by accident.
