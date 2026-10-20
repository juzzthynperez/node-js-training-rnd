# M04 — TypeScript Basics & TypeScript on Node.js

**Week 2 · Days 1–2 · 8 hours**

## 1. Getting Started with TypeScript

### What TypeScript is and why developers use it

TypeScript is JavaScript with a type system that a compiler checks *before* your code ever
runs. You write `.ts` files; the TypeScript compiler reads them, understands what shape
every value is supposed to have, and tells you — line by line — where your code contradicts
those shapes. None of the types exist at runtime. TypeScript's job is to remove whole
categories of bug while you're still typing, not after a user finds them.

Three reasons developers reach for it:

1. **Mistakes surface in milliseconds, not production incidents.** A typo'd property name,
   an argument in the wrong order, a `null` nobody handled — all become compiler errors.
2. **Refactoring becomes safe.** Rename a field on a type, run the compiler, and every
   place that field is used points you at what to fix.
3. **The code documents itself.** A function signature like
   `retry<T>(fn: (attempt: number) => Promise<T>): Promise<T>` tells the next reader
   exactly what goes in and what comes out — no scrolling through the body to find out.

### JavaScript vs TypeScript

TypeScript *is* JavaScript: every valid `.js` program is a valid `.ts` program. The
compiler strips the type annotations and hands plain JavaScript to the runtime. Two
consequences worth internalizing now:

- **Anything JavaScript can do badly at runtime, TypeScript can also do badly** — if you
  ask it to. Types can be escaped (`any` exists; Section 4 covers why to avoid it).
- **The type system only knows what you told it.** It checks the *shapes* of your data,
  not the *meaning* of it. A `title: string` accepts `""`. This distinction —
  type-checking vs runtime validation — is the seed of a later module where real
  validation arrives.

### How static type-checking helps catch mistakes before execution

Given this JavaScript, which runs cheerfully and fails at 2 a.m.:

```js
function fullName(user) {
  return `${user.first} ${user.last}`;
}

fullName({ firstname: 'Ada', lastname: 'Lovelace' }); // undefined Ada
```

TypeScript, with `user` annotated or inferred as an object type, rejects the call site
because `firstname` is not `first`:

```ts
error TS2339: Property 'firstname' does not exist on type
'{ first: string; last: string; }'. Did you mean 'first'?
```

The value of *static* checking is that this happens while the editor still has the
context — the error sits under your cursor, you fix it, it never ships.

### Type-checking vs runtime validation

Keep these two apart from day one:

- **Type-checking** (this module) happens at compile time, catches shape mismatches
  between *your* code and *your* declared types, and disappears entirely when the code
  runs.
- **Runtime validation** checks data arriving from the *outside world* — HTTP bodies,
  environment variables, JSON files — against rules while the program is running.
  TypeScript cannot do this: the outside world never saw your types.

You will meet runtime validation later in the program (with Zod, in Module 8). For now,
the rule is: types check the inside; the outside still needs checking at the boundary.

### Module objectives and deliverables

By the end of this module you can read and write strict TypeScript and set up a TypeScript
project that runs and type-checks on Node.js. The deliverable:

> **The M02 toolkit ported to TypeScript with generics, plus Task-domain types
> (`Task`, `CreateTaskInput`, `UpdateTaskInput`, `Result<T, E>`) — zero
> `tsc --noEmit` errors under a strict config.**

How you'll know you're there: `npm run typecheck` reports zero errors, `npm run toolkit`
passes every self-check, and you can explain `unknown` vs `any` and `type` vs `interface`
in your own words. There is no checklist to fill in — each of those three gets *proved*
as a checkpoint inside the exercises, starting in Section 7 (Section 6 builds the
`typecheck` gate they'll use).

### Reviewing the existing M02 JavaScript toolkit

Everything in this module's lab is a port of code you already trust. Re-read the M02
toolkit (`modules/m02-modern-javascript-async-await/async-toolkit.js`) with an eye on
its *shapes* rather than its behavior — the behavior is proven; your job is to describe
it to the compiler:

```js
export async function sleep(ms) { /* … */ }
export async function retry(fn, { retries = 3, delayMs = 100 } = {}) { /* … */ }
export function withTimeout(promise, ms) { /* … */ }
export async function mapLimit(items, limit, fn) { /* … */ }
```

Four questions to carry through the whole module: what does `sleep` *return*? What does
`retry`'s `fn` *take* and *give back*? What type is the `promise` racing the timeout?
What does `mapLimit` produce from `items`? Plain JavaScript lets these go unanswered;
the port is where you answer each one in writing — as types.

Before you start, set up your workspace. From your `node-js-training-ground` repo root,
check the runtime pin first — this module targets Node 24:

```bash
node --version        # should print v24.x
```

If it prints something older, your shell isn't applying `.nvmrc` (Section 6 explains
which Node this module expects and why). Then start your branch and module folder:

```bash
git switch main
git pull
git switch -c feat/module-04-typescript
mkdir -p modules/m04-typescript-on-node
```

## 2. Understanding the Type System

### Type inference vs explicit type annotations

You don't have to annotate everything — TypeScript *infers* types from what you write:

```ts
let count = 0;        // inferred: number
const names = ['a'];  // inferred: string[]
const total = count + 1; // inferred through the whole expression
```

Annotate when the inference would guess the wrong or too-narrow thing:

```ts
let status: 'pending' | 'done' = 'pending'; // without this, inference gives `string`
let buffer: Uint8Array | null = null;       // without this: `null` alone, useless
```

A working habit: **annotate function boundaries, let inference work inside.** Parameters
and return positions are where a wrong guess silently poisons every caller; local
variables are where inference is almost always right.

### Primitive types, arrays, and tuples

The primitives: `string`, `number`, `boolean`, `null`, `undefined`, `bigint`,
`symbol`, and `void` for "nothing meaningful returned." If you write `let b = null`
with no annotation, it widens (behaves like `any` until assigned) — so annotate when you
mean exactly `null`.

Arrays and tuples:

```ts
const ids: number[] = [1, 2, 3];
const pairs: Array<[string, number]> = [['a', 1], ['b', 2]];
```

An array is "any number of these"; a tuple is "exactly this many, in this order." A
tuple type is what you'd give a `[longitude, latitude]` pair — the two positions mean
different things, and only a tuple says so.

### Object types and structural typing

Object types describe shape:

```ts
type User = { id: string; email: string };
```

TypeScript's typing is **structural** (shape-based), not nominal (identity-based): if a
value has at least the required properties with compatible types, it *is* that type — no
`implements`, no registration. A function expecting `User` accepts any object literal
with `id` and `email`. The edge worth knowing: two unrelated classes with identical
shapes are interchangeable to the compiler even if you meant them not to be.

### Union and intersection types

```ts
type Status = 'pending' | 'in_progress' | 'done'; // union: one OF these
type WithId = User & { id: string };              // intersection: this AND that
```

A union value is *one of its members* — which is what makes narrowing (Section 4)
possible. An intersection stacks shapes; for anything deeper than two fragments, a
named `type` alias reads better.

### Literal types and `as const`

Narrow a type to exact values:

```ts
type Direction = 'up' | 'down';
const OK = 200;   // a `const` keeps the literal 200; a `let` would widen to `number`
```

`as const` freezes an entire structure — properties become `readonly`, values become
literals: `{ readonly retries: 3; readonly levels: readonly ['debug', 'info'] }` for a
config object declared with a trailing `as const`.

This course deliberately uses **union types instead of `enum`** (Section 6 covers the
runtime reason): unions are plain data, they erase completely, and `satisfies`
(Section 5) gives you the checking without the machinery.

### Optional and readonly properties

```ts
interface Task {
  title: string;
  description?: string;   // optional: may be absent (type becomes string | undefined)
  readonly id: string;    // assign-once: mutation is a compile error
}
```

Optional (`?`) and `readonly` are independent: a property can be either, both, or
neither. `readonly` is a *compile-time* guard only — no runtime protection — but it is
enough to stop the accidental `task.id = …` in a codebase you control.

### Choosing appropriate types for data

A small decision procedure for the exercises ahead:

1. Can it be one of several exact words? → union of string literals, not `string`.
2. Can it be absent? → `?` (which unions the field with `undefined`).
3. Does the system, not the caller, own this field? → `readonly`.
4. Is it a fixed sequence of differently-meaning values? → tuple.
5. Is it a list where position matters? → tuple; otherwise array.

Write the type the way you'd explain the data out loud — "a task has a required title,
an optional description, and a status that is one of three words" is already the type.

## 3. Typing Functions and Reusable Code

### Typing function parameters and return values

```ts
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

Parameters are always annotated (inference has nothing to guess *from* — the function
standalone tells the compiler nothing about future arguments). Return types are usually
inferred correctly, but **annotating returns on exported functions is worth the
discipline**: the compiler then *checks* your body against your promise to callers, and
a drifted body is an error, not a surprise.

Callbacks work the same way, and contextual typing often supplies the types inside a
callback for free:

```ts
const numbers = items.map((item) => item.length);
// `item`'s type flows in from `items: string[]` — no annotation needed here.
```

### `type` vs `interface`

Both describe object shapes and overlap heavily:

```ts
type UserT = { id: string; email: string };
interface UserI { id: string; email: string }
```

The differences that matter in practice:

- `interface` can be **declared again and merged** (two `interface Task` blocks combine) —
  useful for extending library types.
- `type` is an **alias for anything**: unions, intersections, tuples, primitives —
  including applications like `Omit<Task, 'id'>`, which an `interface` cannot express.

This module's convention: object-shaped domain models get `interface`; anything the
compiler *computes* gets `type`. There is no performance difference — pick for
expressiveness.

### Introduction to generics

Generics are **type parameters** — placeholders a function keeps in sync between what it
receives and what it returns:

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}

first([1, 2, 3]);      // T = number → number | undefined
first(['a', 'b']);     // T = string  → string | undefined
```

Without the generic, the honest signature would be
`(items: any[]) => any` (no checking at all) or `(items: unknown[]) => unknown` (checked,
but useless to the caller). `T` gives both: checked *and* useful.

### Constraints and default type parameters

**Constraints** restrict which types may fill the placeholder:

```ts
function longestOf<T extends { length: number }>(items: T[]): T | undefined {
  let best: T | undefined = undefined;
  for (const item of items) {
    if (best === undefined || item.length > best.length) best = item;
  }
  return best;
}

longestOf(['a', 'aaa']);        // fine — strings have length
longestOf([1, 2]);              // error — number has no length
```

**Defaults** give `T` a fallback when callers pass nothing:

```ts
type Box<T = string> = { value: T };
let a: Box;         // Box<string>
let b: Box<number>; // Box<number>
```

Constraint says "at least this much"; default says "if unspecified, use this."

### Applying generics to reusable JavaScript functions

Point the generic lens at the M02 toolkit and the signatures write themselves:

- `retry(fn)` — whatever `fn` resolves to is what `retry` resolves to: `retry<T>`.
- `withTimeout(promise)` — the timeout never changes the value type: `withTimeout<T>`.
- `mapLimit(items, limit, fn)` — items are `T`, `fn` turns each into `R`, results are
  `R[]`: `mapLimit<T, R>`.

Signatures to have in front of you as targets during the port (shapes only — you write
the bodies in Section 7):

```ts
export async function retry<T>(
  fn: (attempt: number) => Promise<T>,
  options?: { retries?: number; delayMs?: number },
): Promise<T>;

export async function mapLimit<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T, index: number) => Promise<R>,
): Promise<R[]>;
```

### How generics preserve type information

The point of a generic is that the caller's type *travels through* the function and comes
out the other side. Compare:

```ts
function identityAny(x: any): any { return x; }
const a = identityAny('hi');       // any — checked by nobody, from here on

function identity<T>(x: T): T { return x; }
const b = identity('hi');          // string — `.toUpperCase()` checks, typos caught
```

With `any`, `a` stops being checked the moment it's returned — and `any` *spreads*: one
`any` in the chain disables checking for everything downstream of it. With `T`, the
`string` you passed in is still a `string` when the value comes out. That preservation is
the entire reason `mapLimit<T, R>` can give its caller back a `R[]` instead of `any[]` —
the M02 JavaScript version *couldn't say so*; the port is where you finally do.

## 4. Working with Uncertain Values

### `unknown` vs `any` vs `never`

Three answers to "what is this value?":

| Type | Assignable from | Usable as | Meaning |
| --- | --- | --- | --- |
| `any` | anything | anything | "stop checking" — compiler opts out |
| `unknown` | anything | **nothing until narrowed** | "I don't know yet" |
| `never` | nothing | n/a (it never occurs) | "no value possible" |

`any` is a hole in the type system: it moves freely both directions, and checking stops
around it. `unknown` is the *safe* version of the same flexibility — it accepts anything
but lets you do nothing with it until you prove what it is. `never` is the endpoint of
exhaustion: the return type of a function that always throws, or what's left of a union
after every branch has been handled.

The rule that will serve you all module: **`unknown` + narrowing > `any` + hope.**

### Narrowing with `typeof`, `in`, and equality

Narrowing is how a wide type becomes a specific one *inside a check*:

```ts
function describe(value: string | number): string {
  if (typeof value === 'string') return value.toUpperCase(); // here: string
  return value.toFixed(2);                                   // here: number
}

function move(obj: { kind: 'circle' } | { kind: 'square'; sides: number }) {
  if ('sides' in obj) return obj.sides; // `in` narrows by property presence
}
```

Equality checks narrow too — `if (value === 'pending')` leaves only the literal branch,
and `if (x == null)` covers `null` and `undefined` together (one of the few places
`==` is the right tool).

### Discriminated unions

When every member of a union shares a common literal field — the *discriminant* —
`switch` narrows exhaustively:

```ts
type Action =
  | { type: 'add'; text: string }
  | { type: 'remove'; id: string }
  | { type: 'clear' };

function handle(action: Action): void {
  switch (action.type) {
    case 'add':   console.log(action.text); break;   // narrowed
    case 'remove': console.log(action.id);  break;   // narrowed
    case 'clear': console.log('cleared');   break;   // narrowed
  }
}
```

This pattern — literal tag plus per-member fields — is the most useful shape in this
module: you build one yourself in Section 5 (`Result<T, E>`), and it reappears later
wherever a value can be in one state *or* another. Add a `default` or `never` check and
the compiler refuses to compile when you forget a member — exhaustiveness you can't
skip.

### User-defined type guards

`typeof`/`in` can't check "is this a `Task`". You write the check yourself, and *tell*
the compiler it's a guard with `value is Type`:

```ts
function isTask(value: unknown): value is { id: string; title: string } {
  return (
    typeof value === 'object' && value !== null &&
    'id' in value && typeof (value as { id: unknown }).id === 'string' &&
    'title' in value && typeof (value as { title: unknown }).title === 'string'
  );
}

if (isTask(maybe)) console.log(maybe.title); // narrowed to Task-shaped
```

The `value is …` syntax is a promise — *you* assert what the compiler can't derive. A
guard that returns `true` for non-tasks doesn't weaken the compiler; it makes the
compiler lie for you.

### Parsing an unknown payload end to end

A worked pattern combining everything above — without a single `any`:

```ts
type ParseResult =
  | { ok: true; value: { id: string } }
  | { ok: false; error: string };

function parseId(raw: unknown): ParseResult {
  if (typeof raw !== 'object' || raw === null) return { ok: false, error: 'not an object' };
  if (!('id' in raw) || typeof raw.id !== 'string') return { ok: false, error: 'missing id' };
  return { ok: true, value: { id: raw.id } };
}
```

Every branch is explicit, and a caller of `parseId` who switches on `.ok` gets full
narrowing for free. Notice this is *also* runtime validation — inside a type guard or
parser, you are manually checking the outside world, exactly as Section 1 described.

### Why types alone don't validate runtime data

Everything in Sections 2–4 checks *your* declarations against *your* code. When data
crosses a boundary — `JSON.parse`, an HTTP request, `process.env` — no one on the other
side saw your types, and `JSON.parse` returns `any` **by definition** (annotating its
result changes nothing about the value). The discipline: accept `unknown` at the
boundary, narrow (or validate) it, and only *then* hand typed values onward. Types
police the inside; boundaries are where the manual work happens.

## 5. Modelling Application Data with TypeScript

### Defining the `Task` domain type

The lab's domain: a task tracker. Model it the way Section 2's decision procedure
prescribes:

```ts
export type TaskStatus = 'pending' | 'in_progress' | 'done';

export interface Task {
  readonly id: string;
  title: string;
  description?: string;
  status: TaskStatus;
  readonly createdAt: string;
  readonly updatedAt: string;
}
```

Decisions visible in that block, each defensible: a **literal union** for status (three
known words — not `string`, and deliberately not `enum`); `description` **optional**;
`id`/timestamps **`readonly`** because the system, not the caller, assigns them; and
`interface` for the base object (Section 3's convention) with everything derived below
it as `type`.

### Deriving `CreateTaskInput` and `UpdateTaskInput`

Now the key move — **derive the input types instead of writing them**:

```ts
export type CreateTaskInput = Omit<Task, 'id' | 'createdAt' | 'updatedAt'>;
export type UpdateTaskInput = Partial<Omit<Task, 'id' | 'createdAt' | 'updatedAt'>>;
```

The reasoning, which matters more than the syntax:

1. For each field ask *who assigns it?* The caller supplies `title`, `description`,
   `status`; the system supplies `id`, `createdAt`, `updatedAt` — so the create input is
   `Task` minus the system's fields: `Omit`.
2. A patch allows *any subset* of the caller's fields — `Partial` wrapped around the
   same `Omit`, on the **outside**. (Inside, system fields would merely become optional
   instead of disappearing.)
3. Derivations **stay connected** to `Task`: add `priority?: number` to the interface
   and both inputs pick it up automatically — the compiler enforces what a hand-written
   copy would only *remember*.

### The core utility types: `Partial`, `Required`, `Pick`, `Omit`, `Readonly`

Each is a function over *types*:

```ts
type A = Partial<Task>;    // every property optional — a patch
type B = Required<Task>;   // every property required — strips `?` (description becomes mandatory)
type C = Pick<Task, 'id' | 'title'>;  // only these keys — a summary/projection
type D = Omit<Task, 'id' | 'createdAt' | 'updatedAt'>;  // everything except these
type E = Readonly<Task>;   // every property readonly — freeze at the type level
```

`Pick`/`Omit` are duals (whitelist vs blacklist); `Partial`/`Required` toggle presence;
`Readonly` toggles mutability. If you ever reach for a handwritten duplicate of a type
with two fields changed, one of these six is almost certainly the right tool.

### `Record`, `ReturnType`, and `Awaited`

Three more that show the compiler working *for* you:

```ts
declare function createTask(): Promise<Task>;  // a function that already exists in your code
type Grouped = Record<TaskStatus, Task[]>;  // { pending: Task[]; in_progress: Task[]; done: Task[] }
type R = ReturnType<typeof createTask>;     // Promise<Task> — extracted from the function
type Unwrapped = Awaited<Promise<Promise<string>>>;  // string — peels all promise layers
```

`Record` builds a keyed object from any union — misspell a key and it's an error (and
`Grouped` above is exhaustive over `TaskStatus` by construction). `ReturnType` and
`Awaited` *extract* types from functions and promises, so a signature change propagates
to everything derived from it. Copying a type someone else declared? Ask whether `Pick`,
`ReturnType`, or `Record` can *reference* it instead.

### `satisfies` and `import type`

**`satisfies`** checks a value against a type *without* collapsing its inferred type to
that annotation:

```ts
type Handlers = { onSuccess: () => void; onError: (e: Error) => void };

const handlers = {
  onSuccess: () => console.log('done'),
  onError: (e: Error) => console.error(e),
} satisfies Handlers;
// handlers keeps ITS literal type — and is proven to match Handlers.
// A missing or misnamed member here is a compile error, not a later crash.
```

An annotation (`const handlers: Handlers = …`) would widen everything to `Handlers`;
`satisfies` validates and *preserves* — annotations when the contract should dominate,
`satisfies` when you want both the check and the precise type.

**`import type`** makes type-only imports explicit — and under this module's
`verbatimModuleSyntax` (Section 6), required wherever an import is *only* a type:

```ts
import type { Task } from './task-domain.ts';
import { ok, err } from './task-domain.ts'; // runtime values — real import
```

Type-only imports are erased at compile time; saying so prevents accidentally pulling a
value into the runtime graph and keeps the boundary honest.

### Modelling success and failure with `Result<T, E>`

Not every operation answers with a value — some answer with a *failure*. Instead of
throwing (which the type system can't see), model both outcomes as a discriminated
union:

```ts
export type Result<T, E> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };
```

Why this shape: `ok` is a **literal** (`true`/`false`) — the discriminant — so
`switch (result.ok)` narrows each arm with zero casts, exhaustively. The alternative
`{ value?: T; error?: E }` type-checks *invalid* states (both set, neither set),
reintroducing the bug class `Result` exists to close. Throwing works too, but a thrown
error is invisible to every caller's signature, while `Result<T, E>` *forces* callers to
handle both arms.

### Applying types to practical application scenarios

Where each piece earns its place: a create endpoint accepts `CreateTaskInput` (callers
structurally cannot supply an `id`); a patch endpoint accepts `UpdateTaskInput` (partial
bodies legal, system fields not); a storage function returns `Result<Task, StorageError>`
so callers must handle failure; a grouped view uses `Record<TaskStatus, Task[]>`; a
config checked with `satisfies` fails at compile time if a key is missing. The
through-line: **model what's legal, and illegal states become compile errors instead of
runtime discoveries.**

## 6. Setting Up TypeScript on Node.js

### Project setup: TypeScript 7 and `@types/node@24`

From your module folder (created in Section 1). First a `package.json` — it must exist
*before* the install, or `--save-dev` has nothing to record the dependencies against:

```bash
npm init -y
```

Then install the three dev dependencies:

```bash
npm install --save-dev typescript@7 @types/node@24 tsx
```

Three packages, three jobs:

- **`typescript`** provides the compiler (`tsc`) and the type definitions for the
  language itself.
- **`@types/node`** provides the types for Node's built-ins (`process`, `node:fs`,
  `node:http`…). Without it, `process.env` is not a thing your compiler knows.
  **Match its major version to your Node major** — this cohort runs Node 24 (pinned in
  `.nvmrc`), so `@types/node@24`: the APIs you're told exist are the APIs your runtime
  actually has.
- **`tsx`** — the dev runner used throughout this module: every `npx tsx …` run and the
  `npm run toolkit` script depend on it.

### `tsconfig.json` essentials

The compiler is configured by `tsconfig.json` in the project root. This module's config,
with what each option is doing:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "types": ["node"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["*.ts"]
}
```

What you're looking at, option by option:

| Option | What it does |
| --- | --- |
| `strict` | The master switch — null checks, no implicit `any`, strict function types. Everything else layers on it. |
| `module` / `moduleResolution: nodenext` | Interpret files the way Node 24 does: ES modules, package `exports` honored, **relative imports need explicit `.ts` extensions** — compiler and runtime agree on resolution, so nothing "works in tsc, breaks in Node." |
| `verbatimModuleSyntax` | Imports are emitted exactly as written; a type-only import must say `import type`. |
| `erasableSyntaxOnly` | Only syntax that erases cleanly — the switch that makes Node's native type stripping possible. It outlaws `enum`, `namespace`, and parameter properties: the runtime reason behind "unions, not `enum`" (Section 2). |
| `noUncheckedIndexedAccess` | `items[index]` becomes `T \| undefined` — it will push back during the port (Section 7), so treat every complaint as a real question. |
| `noEmit` | `tsc` only *checks*; it never writes files. (Also what legally unlocks `allowImportingTsExtensions`.) |
| `skipLibCheck` | Don't type-check the inside of dependency `.d.ts` files — big compile-time win, no practical loss. |

Two members of `strict` deserve names: `noImplicitAny` (unannotated parameters are
errors, not silent `any`) and `strictNullChecks` (`null`/`undefined` are their own
types, not shrunk to every other one). If you're ever tempted to switch an option off to
make an error disappear: the error is the point — Section 8 shows how to fix rather
than silence.

### Node.js module configuration

Three rules, all enforced by `moduleResolution: nodenext`:

1. Your `package.json` needs `"type": "module"` — this module's code is ESM (M03 covered
   why).
2. Relative imports carry their extension: `'./task-domain.ts'`, not `'./task-domain'`.
   Node does not guess; the compiler stops you writing something Node can't resolve.
   (`allowImportingTsExtensions` permits the `.ts` form *because* `noEmit` means nothing
   is ever rewritten to `.js`.)
3. Bare specifiers (`'node:assert/strict'`, `'tsx'`) stay as they are — package
   resolution handles them.

### Running TypeScript with tsx

`tsx` is a dev runner: it strips types on the fly (via esbuild) and executes — no build
step, no output files. Edit the `package.json` that `npm init -y` created: set `"type"`
and replace the auto-generated `scripts` block with these three (leave `name`, `version`,
etc. as they are):

```json
{
  "type": "module",
  "engines": { "node": ">=24" },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "toolkit": "tsx async-toolkit.ts",
    "toolkit:native": "node async-toolkit.ts"
  }
}
```

Then `npm run toolkit` runs the file with types removed — it will work as soon as
`async-toolkit.ts` exists (Section 7 ports it). Declaring the scripts now costs nothing,
and `typecheck` works as soon as any `.ts` file exists for the compiler to read. `tsx`
does **not** type-check — it only transforms. That's why `typecheck` is a separate
script; conflating "it runs" with "it's correct" is the classic TypeScript mistake.

### Node.js native type stripping

Node 22+ can execute `.ts` files itself: it *erases* type annotations at load and runs
the result — no dependencies at all. Constraints to know before you rely on it:

- **Erased syntax only.** Non-erasable syntax (`enum`, `namespace`, parameter
  properties) fails with a hard error — which is exactly why this module's config sets
  `erasableSyntaxOnly`, not just for Node's benefit.
- **No type-checking.** Node strips; it never checks. A file that runs natively is
  proof it executes, not proof it's right.
- **Explicit `.ts` import extensions**, same rule as `nodenext`.
- **Maturity varies.** On Node 24 (this cohort) stripping works but is not yet fully
  stable; full stability is expected with Node 26. Run it as an experiment and record
  what *your* attempt prints, warnings included — don't assume a fixed behavior.

To try it, make a tiny `demo-native-strip.ts` in your module folder (one typed line that
prints something is enough), then run it both ways:

```bash
npx tsx demo-native-strip.ts     # dev runner
node demo-native-strip.ts        # Node's own type stripping
```

Identical output is the healthy result — but the two paths are doing very different
work, and Section 8 returns to what each one does and doesn't prove.

### The type-check gate: `tsc --noEmit`

A third runner exists — a real `tsc` build that emits `.js` files into `dist/` (how
libraries ship; awareness-level here, since `noEmit` keeps us honest) — but this
module's gate is one command:

```bash
npm run typecheck      # runs: tsc --noEmit
```

`tsc --noEmit` type-checks the whole project and exits `0` only if there are zero
errors — nothing written, nothing run. As an npm script it's the same command everywhere
(your machine, CI later), and "exit 0" is a machine-checkable green light. Run it early,
run it often — every file you add, before every commit.

### Typing async code with `Promise<T>`

`Promise<T>` is just a generic like `retry<T>` — the value type *inside* the box:

```ts
async function loadTask(id: string): Promise<Task> { /* … */ }
const task = await loadTask('t1'); // task: Task
```

Worth knowing: `Promise<void>` for async functions with no value in the resolution;
`Promise.all([...])` infers a tuple of the resolved types; `await` unwraps one layer
only (hence `Awaited<T>`, Section 5). A `Promise<unknown>` puts you back in Section 4:
narrow before using it.

### Typing caught errors with `unknown`

Under `strict`, the variable in `catch (e)` is **`unknown`** — TypeScript won't let you
pretend otherwise (the only annotations a catch clause accepts are `any`/`unknown`, and
`any` is the mistake this section exists to prevent). The reason it defaults to
`unknown`: *anything* can be thrown — an `Error`, a string, a number, an object from
library code that never read the rules. So:

```ts
try {
  await risky();
} catch (error) {
  if (error instanceof Error) console.error(error.message);
  else console.error(String(error));
}
```

The pattern — narrow, then read members — is the same discipline as Section 4. It will
show up the moment your ported self-check harness reads `.message` off a caught value.

### Typing environment variables, `@types/*` packages, and `.d.ts` files

`process.env` is typed as `{ [key: string]: string | undefined }` — every key
*possible*, values **always strings**. So missing variables are `string | undefined`
(narrow or fail loudly — never let `undefined` flow silently into config), and
numeric-looking values still need `Number(...)`.

The ecosystem, briefly: **`@types/*`** packages (DefinitelyTyped) type *other people's*
JavaScript — `@types/node` types the runtime; if a package ships its own types, don't
install `@types` for it. **`.d.ts` files** are the declaration-file format (types with
the implementation erased) — what every `@types` package is made of, and what `tsc`
consults.

## 7. Porting the M02 Toolkit to TypeScript

### The port's contract: review behaviors before touching code

You reviewed the shapes in Section 1; now review the *behaviors* you must preserve:
**same runtime behavior, better static description.** Every M02 self-check that passed
in JavaScript must pass in TypeScript — the port may not "improve" anything except what
the compiler can see.

### Set up the project before porting any code

Order matters, and it's the lab's first rule: **project first, port second.** Before a
single function is touched:

1. `package.json` with `"type": "module"` and the scripts from Section 6.
2. The dev dependencies installed (`typescript`, `@types/node@24`, `tsx`).
3. `tsconfig.json` with the strict config from Section 6.
4. `npm run typecheck` exiting 0 on a **starter** project — any minimal `.ts` file
   suffices (`tsc` errors TS18003 if the config matches zero files, so don't run it on a
   literally empty folder).

A green baseline means every later error is attributable to a line you just wrote, not
to a config problem hiding three files away. Smallest possible green, then add — this
habit generalizes to every project after this module.

### Porting `sleep` and `withTimeout`

Start with the easy wins to prove the pipeline (`tsc --noEmit` → tsx → run):

- `sleep(ms)` gains `ms: number` and `Promise<void>`; keep its `RangeError` guard
  exactly — behavior, not types.
- `withTimeout(promise, ms)` becomes `withTimeout<T>(promise: Promise<T>, ms: number):
  Promise<T>` — one generic passed straight through. `TimeoutError`'s class needs a
  declared field (`readonly ms: number`) before the constructor assigns it.

**Checkpoint:** `npm run typecheck` reports no errors for these two, and the
sleep/timeout self-checks pass under `npm run toolkit`.

### Porting `retry<T>`

Requirements, in the order they'll bite:

1. Generic signature: `retry<T>(fn: (attempt: number) => Promise<T>, options?):
   Promise<T>` — options bag typed inline or as a named interface, defaults preserved.
2. The option-validation `RangeError`s stay verbatim — behavior, not types.
3. `catch (error)` gives you `unknown`. Your loop remembers the last failure to rethrow
   it — declare that variable as `unknown`, **not** `any`. Verified surprise: TS2454
   (used before assigned) does *not* fire for it, because `undefined` inhabits
   `unknown`, so "possibly unassigned" is still in-type. Prove it: same variable typed
   `string` → TS2454 appears; typed `unknown` → it vanishes. Either way: no assertion
   (`!`), no defensive seed. A compiler-behavior claim deserves the two-line repro —
   believing without it is how wrong comments get written.
4. The rethrow at the end must rethrow the *last* error unchanged.

**Checkpoint:** typecheck is clean, and the retry self-checks (success after failures,
exact attempt count, rethrow, option validation) all pass.

### Porting `mapLimit<T, R>`

The interesting one. Requirements:

1. Signature: `mapLimit<T, R>(items: readonly T[], limit: number, fn: (item: T,
   index: number) => Promise<R>): Promise<R[]>` — note `readonly T[]`: the function
   never mutates its input, and the type says so.
2. Preserve every behavior: input-order results, the concurrency ceiling, fail-fast
   rejection, empty-input handling, invalid-limit `RangeError`.
3. **The `noUncheckedIndexedAccess` confrontation.** Your M02 body reads
   `items[index]` inside a `index < items.length` loop. The compiler will now tell you
   that read is `T | undefined` — and that your loop condition does *not* prove
   otherwise (it has no idea they're related). Three ways out: `items[index]!` (assert
   the invariant the compiler can't see), a `!== undefined` guard (which silently skips
   any element that legitimately *is* `undefined`), or **restructure so the type carries
   the guarantee**. Prefer the third — a shared `Symbol.iterator` pull is the shape that
   makes `step.done === false` narrow to `T` honestly. Whatever you choose, the
   reasoning goes in your notes: this is the module's central lesson about expressing
   invariants in *types* rather than comments.

**Checkpoint:** the order, concurrency-ceiling, fail-fast, and error-propagation
self-checks all pass and typecheck is clean — with **no** `any`, and no `!` you can't
justify out loud.

### Resolving type errors incrementally

When typecheck reports an error, fix the *code*, not the *compiler*. Read the error end
to end (Section 8 teaches the anatomy), find the line it names, decide whether the code
or the annotation is wrong. One error at a time — downstream errors often evaporate when
the root cause is fixed, so re-run typecheck after every fix rather than working a queue.

### Running the toolkit with tsx

```bash
npm run typecheck
npm run toolkit
```

Both green = the deliverable's core gates. Record the self-check count you actually see
— the expected property is *all of them passing*, not a specific number.

### Why TypeScript improves the maintainability of reusable code

Compare the M02 and M04 versions of the *same* functions: callers of `mapLimit` get
`R[]` instead of an unchecked array; callers of `retry` get their `T` back; wrong
argument shapes stop at your keyboard instead of at runtime. The bodies barely changed —
the *contracts* got real. That ratio (tiny implementation diff, large contract diff) is
why TypeScript pays off most on small utilities everyone imports, where one silent
mistake propagates everywhere at once.

## 8. Type-checking, Debugging, and Verification

### Reading compiler errors

Anatomy of the errors you'll actually meet:

```text
async-toolkit.ts(270,20): error TS18046: 'error' is of type 'unknown'.
```

Read it as four beats: **where** (`file(line,col)`), **what class**
(`error TS18046` — the number is searchable; the docs page for that code is the fastest
deep-dive on your problem), **the verdict** ("is of type `unknown`"), and often a
**hint** (TS2339's "Did you mean 'first'?" is usually right — take the hint).

The errors this module throws at you most:

| Code | Means | Fix direction |
| --- | --- | --- |
| `TS18046` | value is `unknown` | narrow it (Section 4) before use |
| `TS2322` | assigned type doesn't match declared | fix the value or re-examine the annotation |
| `TS2339` | property doesn't exist on type | check spelling; then check the *shape* |
| `TS2532`/`TS2531` | possibly `null`/`undefined` | narrow or handle the empty case |
| `TS2454` | used before assigned | restructure flow or seed — not `!` by default |

Panic-killer: **the compiler is describing a contradiction, not a catastrophe.** It
cannot delete your code; it's telling you exactly which two lines disagree.

### Fixing type errors without unnecessarily suppressing them

- **Fix:** make the code actually correct — narrow the `unknown`, correct the
  annotation, handle the `undefined` case.
- **Suppress:** `as any`, `!`, `@ts-expect-error`, turning an option off — each a signed
  statement that "the compiler is wrong here", each making the next real bug better
  hidden.

A justified `!` (you can prove the invariant) is legitimate; a `!` added to stop an
error you didn't understand borrows time at compound interest. The test: *if this code
had a bug, would this suppression hide it?* If yes, fix instead.

### Running `npm run typecheck` as a habit

Run it after every file and **before every commit**. It's cheap (TS 7's native compiler
makes it cheaper still), and a clean run on commit means the branch you push is green by
construction — exactly what a later CI job will demand of the same code.

### Understanding the role of strict compiler options

Each strict option closes a specific escape hatch: `strictNullChecks` ends "it can't be
null… can it?"; `noUncheckedIndexedAccess` ends "that index is fine, trust me";
`noImplicitAny` ends the silent spread of `any`. Turning one off doesn't remove the
*risk* — it removes the *detector*. If a strict option makes code impossible, the code's
model of the data is usually wrong, and the error is doing its job.

### Attempting native type stripping on Node 24

Run it and **observe**:

```bash
node async-toolkit.ts            # or: node demo-native-strip.ts
```

Whatever happens — full success, an experimental warning, or a hard
`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` — note it exactly, with your `node --version`: that's
what your write-up quotes. The contrast to draw: this command does **no type-checking
whatsoever** — it proves the file *executes*, nothing more. Only `tsc --noEmit` proves
the types are right. (On Node 24 it's awareness-level; full stability arrives with
Node 26 — don't build a workflow on it yet.)

### Comparing `tsx`, native type stripping, and a compiled `tsc` workflow

Three runners, three jobs — keep them straight:

| Runner | Does it type-check? | Speed | Use it for |
| --- | --- | --- | --- |
| `tsx` | **No** (pair it with `tsc --noEmit`) | instant | daily development |
| `node file.ts` (stripping) | **No** | instant | quick runs, awareness of the platform path |
| `tsc` build → run `.js` | **Yes** (it's the compiler) | build step | shipping, libraries, non-erasable needs |

The trap to avoid: "it ran" ≠ "it's type-safe." Only `tsc --noEmit` answers the second
question, which is why the *typecheck* script — not a run — is the module's gate.

### TypeScript 6 → 7: what changed (introductory)

So you recognize the names later: TypeScript 7 is the **native compiler** (rewritten
for speed — that's why `typecheck` is fast), **strict by default** for new projects, and
**legacy options were removed** rather than silently honored. It also has **no stable
programmatic API**, which is why some editor/tool integrations had rough edges this era
— you'll meet that consequence again in a later module. Nothing to act on now.

## 9. Hands-on Practice and Verification

The lab is verification, not discovery — build nothing new here; prove what Sections
5–8 produced. Work through it in order: each step ends with the run that proves it.

### Complete the Task-domain types

Requirements (in your project, `task-domain.ts` or equivalent):

1. A `TaskStatus` literal union and a `Task` model with `readonly` system fields and an
   optional field — following the reasoning in Section 5, not just its shapes.
2. `CreateTaskInput` and `UpdateTaskInput` **derived** with `Omit`/`Partial` — no
   hand-written field lists. Add a field to `Task` afterward and confirm both inputs
   follow automatically (that check is the proof the derivation is real).
3. Demonstrate at least three *other* utility types from Section 5 on your domain
   (`Pick`, `Record`, `Required`, …) with a one-line comment each saying why that one.

**Checkpoint:** `npm run typecheck` stays clean, and you can point at each derivation and
say who owns each field.

### Implement the discriminated-union `Result<T, E>`

Requirements: the two-arm union discriminated on `ok`; `ok(value)`/`err(error)`
constructors; at least one user-defined type guard (`isOk`) written with the
`result is …` syntax; and a function whose `switch (result.ok)` handles both arms
exhaustively — plus a moment where you *deliberately* break exhaustiveness (add an
unhandled arm) and watch the compiler refuse.

**Checkpoint:** the `switch` on `.ok` narrows without any casts, and the compiler objects
when you stop handling a case.

### Verify the ported toolkit

```bash
npm run typecheck        # zero errors
npm run toolkit          # every self-check passes via tsx
```

If any self-check fails, it's a behavior regression in the port — go back to Section 7's
requirements, don't edit the checks to match the code. Keep both runs' output handy; you
quote them in your write-up.

### Attempt native type stripping

```bash
node --version           # your version
node async-toolkit.ts    # exactly what happens
```

Success, warning, or hard error — copy the literal output into your notes. Then run the
demo both ways and confirm they agree:

```bash
npx tsx demo-native-strip.ts
node demo-native-strip.ts
```

### Before you close the module

One last back-to-back run of both gates — `npm run typecheck` and `npm run toolkit` —
green. Then the honest check: say out loud, no notes, what `unknown` does that `any`
doesn't, and why `Result` uses a literal `ok` rather than two optional fields. If an
answer stalls, that's a signpost back to Sections 4–5, not a failure — re-read, then run
the gates once more.

## 10. Wrap-up and Self-assessment

### Explain `unknown` vs `any` in your own words

Three or four sentences, no notes: what each type lets you do, what it *stops* you from
doing, and why `catch` gives you the safe one. If your explanation includes the word
"trust," you've got it — `any` asks the compiler to trust you; `unknown` asks you to
earn it by narrowing.

### Explain `type` vs `interface` in your own words

Same rules: your own words, not Section 3 recited. Anchor it in a decision you actually
made — why `Task` was one and `Result`/`CreateTaskInput` the other. Test: which can
express `'a' | 'b'`, and why can't the other?

### Explain how generics improve the toolkit functions

Pick `mapLimit<T, R>` and narrate the type's journey: `T` in, `R` out through `fn`,
`R[]` back to the caller — and one sentence on what the M02 JavaScript version could
*not* say to its callers. Then do the same for `retry<T>` in a sentence or two.

### Explain what the TypeScript compiler checks and what still requires runtime validation

The module's big idea, stated back: the compiler checks *your code against your types* —
shapes, exhaustiveness, nullability — before execution, erased at runtime. It does
**not** check data from outside (JSON, HTTP, env vars), doesn't check your types are
*true* (a guard can lie), and doesn't check logic. Boundaries need runtime validation —
you met the manual version in Section 4's parser; the framework version comes later.

### Summarize key lessons and challenges

Your own, but a scaffold if it helps: (a) the inference-vs-annotation habit you settled
on; (b) the strict option that fought you hardest and what it taught you
(`noUncheckedIndexedAccess` is the usual winner); (c) one compiler error whose *message*
you now read differently than the first time you saw it.

### Complete the M04 write-up

Write the write-up while the experience is fresh: what you built, the decisions and why
(the utility types you chose, how you modeled `Result`, what your native-stripping
attempt actually printed), the concept you'd explain to the next person taking this
module, what tripped you up, your checkpoint evidence, what you'd do differently. Write
it so someone starting this module next cohort could follow your path alone — that
constraint is what turns notes into a usable record.

### One last run

`npm run typecheck` and `npm run toolkit`, both green, one more time — the module's
claims are demonstrated, not asserted. That's the module; move on to the next stage.
