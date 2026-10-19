# M02 — Modern JavaScript & Async/Await

**Week 1 · Days 2–4 · 10 hours**

## 1. Getting Started

### What you'll learn and build

By the end of this module you will be able to:

- Read and write modern JavaScript confidently: `const`/`let`, closures, destructuring,
  optional chaining, the new array helpers, classes with private fields, and custom errors.
- Explain, in your own words, what happens between "I called an async function" and "its
  result appeared": the call stack, the event loop, and the task vs microtask queues.
- Write reliable asynchronous code with promises and `async`/`await`, including combining
  parallel work safely and handling failures without losing them.
- Build the four utilities of the **Async toolkit** — `sleep`, `retry`, `withTimeout`,
  `mapLimit` — each proven by its own `node:assert` self-check.
- Measure and explain the difference between sequential and parallel work against a real
  public API.

**Expected outcome:** a working toolkit module, passing self-checks, real measured
timings, a refactored callback example, and a write-up of your own experience — plus the
ability to predict the output order of a mixed `setTimeout`/promise/`await` snippet and
say *why*.

### Connecting M02 to the skills learned in M01

M01 gave you the toolchain: a terminal that answers, `node` that runs a file, and the
**branch → commit → PR → merge** habit. M02 reuses all of it — every build step here ends
in the same workflow you practiced.

### Project setup and tool requirements

- **Plain JavaScript only.** No TypeScript in Week 1 (that arrives in M04).
- **ES modules** (`import`/`export`) — no CommonJS `require` in this module.
- **`node:assert` for self-checks** — no test framework yet (that is M09). A self-check
  is a small script that throws if behavior is wrong; `node` exits non-zero when it
  throws, which makes it a real pass/fail gate.
- **Node 24 LTS** (pinned for the cohort), a text editor, a terminal, and network access
  to `jsonplaceholder.typicode.com` (free, no API key).
- **Free tools only**; all code lives in the training-ground repo under
  `modules/m02-modern-javascript-async-await/`.

> **Check your Node first.** Run `node --version`. If it does not say `v24.x`, your
> version manager has not applied the repo's `.nvmrc` — with fnm, either start the shell
> in the repo folder or run via `fnm exec --using=24 -- node your-file.js`. Never assume
> the config file made it true; ask the tool.

### Your goal: build a working Async Toolkit

The deliverable is **one ES module** containing four small utilities that are easy to get
*subtly* wrong:

| Utility | Signature | The trap it hides |
| --- | --- | --- |
| `sleep` | `sleep(ms)` | Timing checks that lie about how long they waited |
| `retry` | `retry(fn, { retries, delayMs })` | Swallowing the final error, or not actually delaying |
| `withTimeout` | `withTimeout(promise, ms)` | Creating *unhandled* rejections while racing a timeout |
| `mapLimit` | `mapLimit(items, limit, fn)` | Exceeding the limit, losing input order, or vanishing errors |

Each utility ships with a `node:assert` self-check **in the same file**. The lab also
includes a sequential-vs-parallel timing comparison against JSONPlaceholder and a
callback-style `node:fs` routine refactored to `async`/`await`.

You'll work one loop all module long — this module adds it *inside* the M01 branch
workflow:

1. **Research** the concepts a build step needs — *before or alongside* the build, never
   after. You cannot look things up honestly once the code already works and you just
   want the box ticked. Sections 2–4 are your research.
2. **Build** one small piece.
3. **Verify** with an executable check (`node:assert`) or a measurement — "it seems
   right" is not verification.
4. **Document** what happened — decisions, surprises, numbers — while they are fresh.

---

## 2. Writing Modern JavaScript

This section is your language research: work through it with a scratch file and tick the
research boxes in `tasks.md` only when you've actually run the example, not when you've
read it.

### Variables, scope, closures, arrow functions, and `this`

`const` is the default: a binding you will not reassign. `let` when the value must change
(loop counters, accumulators). `var` is legacy — function-scoped and hoisted in confusing
ways; do not use it.

```js
const limit = 3;      // limit = 4  →  TypeError
let attempts = 0;     // attempts++ → fine
```

**Scope** is the region where a name is visible: block scope (`{ }`), function scope,
module scope. **Closure** is a function remembering the variables from where it was
*created*, even after that scope has finished running:

```js
function makeCounter() {
  let count = 0;               // this variable lives on...
  return () => ++count;        // ...inside the returned function
}
const next = makeCounter();
next(); // 1
next(); // 2                   // the count survived makeCounter() returning
```

Closures power every callback you write — and they are why a callback can still see
`limit` long after the surrounding function returned.

```js
const double = (n) => n * 2;            // implicit return, no `() => { return }` needed
```

Arrows do two special things: they **return implicitly** (single expression) and they
**do not create their own `this`** — they inherit it from the surrounding function. That
second rule makes them ideal as callbacks (`items.map((n) => n * 2)`) and wrong as object
methods when you need the object as `this`.

### Template literals, destructuring, spread/rest, and default parameters

```js
const describe = (name) => `hello ${name}`;  // template literal: `${}` interpolates
```

```js
const { retries = 3, delayMs = 100 } = options;   // defaults when undefined
const [first, ...rest] = items;                    // rest gathers the remainder
const copy = { ...original, retries: 5 };          // spread copies into a new object
const show = (name, { loud = false } = {}) => {}; // destructured param + its own default
```

Common mistakes: destructuring a property that is `undefined` (defaults only apply to
`undefined`, **not** to `null`), and mutating the object you spread instead of copying it.

### Optional chaining, nullish coalescing, truthy/falsy values, and equality

```js
const ms = config?.timeout?.value;   // undefined instead of TypeError if anything is nullish
const n = maybeNull ?? 0;            // only ??/undefined/null fall through — 0 and '' survive
settings.retries ??= 3;              // assign ONLY if currently null/undefined
```

The trap people fall into: using `||` for defaults and silently losing falsy-but-valid
values — `retries || 3` turns a deliberate `0` into `3`. Use `??`.

Falsy: `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN` — **everything else is
truthy**, including `'0'`, `[]`, and `{}`.

```js
if (value) { /* runs for 'hello', [], {} — did you mean "non-empty"? */ }
```

`==` coerces (`'1' == 1` → `true`), `===` compares value *and* type. Rule of thumb:
default to `===`; reach for `==` only when you genuinely want its coercion documented in
a comment. And remember: `NaN === NaN` is `false` — use `Number.isNaN(x)`.

**Common mistakes and practical examples**

- Reassigning a `const` binding (the *binding* is fixed; the object it points at can
  still change).
- Using `forEach` where you need a result — it returns `undefined`. Use `map`.
- `if (str.length)` on a possibly-`undefined` string → TypeError. Use `str?.length`.
- `value || default` when `value` can legitimately be `0` or `''` → use `??`.

> **Try it:** write a function `describeUser({ name, plan = 'free', credits = 0 })` that
> returns `` `${name} (${plan}, ${credits} credits)` ``, and one test call for a user with
> `credits: 0` — confirm the `0` survives.

### Array methods and non-mutating array helpers

```js
const ids = [1, 2, 3];
ids.map((id) => id * 2);            // [2, 4, 6]        — transform, same length
ids.filter((id) => id > 1);         // [2, 3]           — keep some, maybe fewer
[{ tags: ['a'] }, { tags: ['b'] }]
  .flatMap((o) => o.tags);          // ['a', 'b']       — map + flatten one level
```

All three are **non-mutating**: they return new arrays. `flatMap` is the fix for the
classic `map` + `flat` two-step (and for building one-to-many mappings without nested
arrays).

```js
users.find((u) => u.id === 7);      // the item or undefined
users.some((u) => u.admin);         // at least one?   → boolean
users.every((u) => u.active);       // all of them?    → boolean
```

`find` returns an **element**; `some`/`every` return a **boolean**. Mixing them up is the
most common mistake in this group (`if (arr.find(...))` "works" until the element is
falsy — prefer `arr.some(...)` for existence).

```js
const total = prices.reduce((sum, price) => sum + price, 0);
```

The four-arg callback: `(accumulator, item, index, array)`. Always pass the initial value
— without it, `reduce` on an empty array throws, and the first item is silently used as
the seed. `reduce` builds anything: sums, counts, maps, even your own `group by`.

```js
const sorted = nums.toSorted((a, b) => a - b);  // copy, original untouched
const flipped = nums.toReversed();              // copy, reversed
const patched = nums.with(0, 99);               // copy with index 0 replaced
```

These exist because `sort`, `reverse`, and `arr[i] = x` **mutate in place** — a hidden
bug factory when two functions share one array. New rule: `to*` versions for copies,
originals only when you mean to change them.

### `structuredClone`, object grouping, `Map`, and `Set`

```js
const backup = structuredClone(state);   // deep copy: nested objects/arrays are new too
```

Spread (`{ ...obj }`) copies **one level** — nested objects are still shared references.
`structuredClone` goes all the way down (no functions, class instances, or DOM nodes
allowed — it throws on those).

```js
const byPlan = Object.groupBy(users, (u) => u.plan);
// { free: [...], pro: [...] }
const byPrefix = Map.groupBy(words, (w) => w[0]);   // when keys aren't strings
```

Compare with the manual `reduce`-into-an-object dance: one line, no accumulator bug.
`Object.groupBy` produces a null-prototype object (safe even when words are `'constructor'`
etc.); use `Map.groupBy` when keys aren't strings.

> **Try it:** take `const sales = [{ region: 'EU', amt: 10 }, { region: 'US', amt: 5 },
> { region: 'EU', amt: 3 }]` and compute total-per-region **three ways**: `reduce`,
> `groupBy` + a `map` over entries, and `flatMap` + `filter`. Same answer, three shapes.

| Structure | Use when | Key/lookup notes |
| --- | --- | --- |
| Plain object | Fixed shape, string keys (config, JSON data) | Keys must be strings/symbols; no size |
| `Map` | Dynamic key→value lookups, insertion order matters | Any key type, `.size`, iterable directly |
| `Set` | Unique values, membership tests | `has()` is O(1); no duplicates by design |

Objects are still right for data you will `JSON.stringify`. Reach for a `Map` in code
like a cache or a tally — `seen.has(id)` reads clearer than `Object.hasOwn(seen, id)` and
keys can be anything (even objects or functions).

```js
const a = new Set([1, 2, 3]), b = new Set([2, 3, 4]);
a.union(b);             // Set {1, 2, 3, 4}
a.intersection(b);      // Set {2, 3}
a.difference(b);        // Set {1}
a.symmetricDifference(b); // Set {1, 4}
a.isSubsetOf(b);        // false
```

They all return **new Sets** — the set methods don't mutate, unlike their math-textbook
namesakes in some other languages. Classic uses: unique merge of two tag lists, "who
answered everything", removing known-bad ids.

### Classes, private fields, getters/setters, and error handling

```js
class RetryPolicy {
  #attempts = 0;                       // truly private: invisible outside the class body
  constructor(max) { this.max = max; }
  get attempts() { return this.#attempts; }   // read like a property...
  record() { this.#attempts++; }              // ...but backed by real encapsulation
}
```

`#field` is enforced by the engine (not the old `_convention` handshake). Getters let you
derive a value on read (`get fullName()`); setters let you validate on write
(`set retries(v) { if (v < 0) throw ... }`). Watch out: a getter that reads a field you
also mutate inside the getter will recurse — and destructuring an object **invokes**
getters, while spreading `{...obj}` copies their *current* values.

```js
try {
  await withTimeout(fetchSomething(), 200);
} catch (error) {
  console.error('failed:', error.message);   // handle, translate, or rethrow — don't vanish
} finally {
  closeThing();                              // runs on success AND failure: cleanup
}
```

Rules of thumb: `catch` must do *something* (log, translate, rethrow) — an empty one is a
program that pretends nothing happened. `finally` is for cleanup that must run either way
(closing files, clearing timers, releasing locks).

```js
class TimeoutError extends Error {
  constructor(ms, options) {
    super(`Promise did not settle within ${ms}ms`, options);
    this.name = 'TimeoutError';   // class name is NOT set automatically
    this.ms = ms;                 // carry structured data on the error
  }
}
```

Why not `throw new Error('timeout')`? Because callers must **branch on meaning**, and
matching English strings (`/timed out/i`) breaks the first time you reword the message.
`error instanceof TimeoutError` is a contract; a message is not. Always extend `Error`
(directly or through your own base) — extending nothing makes `instanceof Error` false and
loses the stack.

```js
try {
  await fetch(url);
} catch (error) {
  throw new Error(`could not load ${url}`, { cause: error });
}
```

The outer error says *what task failed*; `error.cause` keeps *why* (the original
`TypeError`/`ECONNREFUSED`) intact for logs. Catching and rethrowing **without** `cause`
is how root causes disappear — you keep only the wrapper's story. Print causes when you
debug: `console.error(err.cause ?? err)`.

> **Try it:** write a `class ValidationError extends Error` with a `field` property, throw
> it from a function, and catch it two ways: `instanceof` branch, then print
> `error.cause` after wrapping it with `new Error('save failed', { cause: error })`.

### ES modules and named vs default exports

A module is a file with `import`/`export` — its own scope. Nothing leaks out unless you
export it, which turns file boundaries into API boundaries. Two side effects worth
knowing: modules are **deferred** (parsed before the rest of the page/main script runs)
and **cached** (import the same file twice — one instance, top-level code runs once).
Top-level code running on import is a feature (shared state) and a footgun (side effects
you didn't mean to trigger — see the entry-point guard in §5).

```js
export function sleep(ms) { /* ... */ }      // named: explicit, renameable, tree-shakable
export default function sleep(ms) { /* ... */ } // default: one main thing per file
import sleep from './sleep.js';              // default: name is chosen by the importer
import { sleep, retry } from './toolkit.js'; // named: names are fixed by the exporter
```

Rule of thumb: **named exports** for libraries of utilities (discoverable, explicit,
`import { retry }` documents the dependency) — this module's toolkit uses named exports
only. A default export suits a file whose single main product is one thing (a component,
a router). Mixing both invites confusion about what a file "is".

ESM path rules that trip beginners: **relative paths need `./`** (`'./x.js'`, never
`'x.js'` — bare specifiers are for packages), **extensions are required** (no Node-style
extension guessing), and Windows accepts `/` in specifiers even though paths are `\`.

> **Try it:** create `scratch.js` with `import { sleep } from './async-toolkit.js'`,
> call it, and run it. Then remove the import and add `export const x = 1` — confirm you
> now get a "must use import" or extension error when you forget `./`. Breaking module
> rules once teaches them faster than reading them.

---

## 3. Understanding Asynchronous JavaScript

This section builds the mental model you'll narrate out loud at the checkpoint: what runs
when, and why.

### Synchronous vs asynchronous execution

**Synchronous:** do this, wait, then do the next thing — one call stack, in order.
**Asynchronous:** start this, *don't wait here* — do other work, and the finished thing's
continuation gets queued. JavaScript itself is single-threaded; the *waiting* happens
elsewhere (timers, network, disk), and results come back through queues.

### Call stack, event loop, task queue, and microtask queue

The **call stack** is where function calls live while they run — one entry pushed, runs,
popped. It can only do one thing at a time: a synchronous infinite loop freezes
everything, because the **event loop** never gets a turn. The event loop is the
while(true) underneath it all: *run sync code to completion → drain microtasks → run one
task → repeat.*

```js
console.log('1');
setTimeout(() => console.log('3'), 0);
console.log('2');
// prints 1, 2, 3 — the callback can't print '3' until the stack empties
```

Two queues, strictly ordered:

- **Microtasks:** promise `.then`/`catch`/`finally` callbacks, `await` continuations,
  `queueMicrotask`. Drained **completely** after every run of sync code.
- **Tasks (macrotasks):** `setTimeout`, `setInterval`, I/O, UI events. **One** per trip
  around the loop, and only when microtasks are empty.

Priority in one line: *sync code > all queued microtasks > one task > all queued
microtasks > …*

- `setTimeout(fn, 0)` means "next trip around the loop", **not** "immediately" — every
  already-queued microtask runs first.
- `Promise.resolve().then(fn)` runs in the **current** microtask drain — before any
  pending timeout.
- `await expr` pauses the async function and schedules its continuation as a
  **microtask** — it does not block the stack; everything around it keeps running.

### Callbacks, callback hell, promises, and `async`/`await`

A **callback** is "call this function when you're done": `fs.readFile(path, (err, data)
=> { ... })`. Node's original async style — error-first: first callback argument is `err`
or `null`.

```js
fs.readFile('notes.txt', (err, data) => {
  if (err) { console.error(err); return; }
  console.log(data.toString());
});
```

Not the indentation — the **semantic** problem: each nesting level must:

- carry its own `if (err)` (miss one and the error vanishes),
- close over everything it needs (deep closures),
- and cannot be composed (no way to say "wait for these three, then…").

When error handling is manual at every level, reliability is a discipline instead of a
property — that's what the refactor in §7 makes visible.

A promise is a value that *will arrive*: pending → fulfilled (value) or rejected
(reason). Creating one wraps work whose completion triggers settle:

```js
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```

Consuming = `.then(onFulfilled, onRejected)` (or `.catch`, `.finally`). Rules: a promise
settles **once** (later resolve/reject calls are ignored), and handlers never run
synchronously — always in a future microtask.

`async` is sugar over exactly that machinery: the function always returns a promise, and
`await` unwraps one (pausing only that function, scheduling continuation as a microtask).
Inside an `async` function everything reads top-to-bottom like sync code.

```js
async function getPost(id) {
  const post = await fetchPost(id);     // throws on rejection, instead of .catch plumbing
  return { ...post, fetchedAt: Date.now() };
}
```

Two rules that save hours: `await` only exists **inside `async` functions** (a top-level
`await` works in modules, but not inside a plain callback), and `await` unwraps the value
*or throws the rejection* — which is why `try/catch` works on it.

### Promise chaining and asynchronous error handling

```js
fetchPost(1)
  .then((post) => fetchComments(post.id))
  .then((comments) => console.log(comments))
  .catch((error) => console.error('chain failed:', error.message))
  .finally(() => loading = false);   // runs on success AND failure — cleanup
```

Each `then` returns a **new** promise, which is why chains flow. Placement of `catch`
matters: at the end it sees failures from *any* earlier link; in the middle it *handles*
them and the rest of the chain continues as success. `finally` passes its value through
unchanged — it's for cleanup, not transformation.

Rejections propagate like thrown errors: an unhandled rejection in a chained promise
travels to the next `.catch`/`try` in the chain — and if there is none, it reaches
`process.on('unhandledRejection')` (by default: warning + in newer Node, a crash). The
async mental model: **`await` + `try/catch` is `then/catch` with better control flow.**

```js
try {
  const a = await step1();
  const b = await step2(a);   // if step2 throws, step1's result is simply done — no pyramids
} catch (error) {
  report(error, { cause: error.cause });
}
```

Key difference from callbacks: in a chain you must *decide where the catch goes*; with
`await` the failure lands exactly where you put `try`. And remember: calling an async
function **without** awaiting it gives you a floating promise — errors inside it are no
longer in your `try`.

> **Try it:** convert this pyramid to `async`/`await` and prove both handle the failure
> identically: `readFile(→ parse(→ save(, with an error injected in `parse`.

### Predicting the output order of `setTimeout`, promises, and `await`

A reliable procedure (practice it until it's automatic):

1. Draw one line for the synchronous pass — log everything that runs without waiting.
2. Whenever you *see* a `.then(...)` or an `await` being registered, write down what queue
   job it queues and in what order.
3. Drain the microtask queue in FIFO order — note that handlers queue *new* microtasks
   behind whatever is already waiting.
4. Take **one** task from the task queue, back to step 2.

Aim for sentences like: *"The `setTimeout` printed last because its callback is a task,
and tasks only run after the entire microtask queue — including the continuation queued
by `await` — is empty."* If you can't say *why* an order happened, the prediction was a
guess. Being able to narrate the queue states out loud is the module's core checkpoint —
it's what separates "I know it prints 1-2-3" from "I understand the event loop".

> **Try it:** predict, write it down, then run:
> ```js
> console.log('start');
> setTimeout(() => console.log('timeout'), 0);
> Promise.resolve().then(() => console.log('micro'));
> console.log('end');
> ```
> Confirm `start, end, micro, timeout` — then explain the `end` vs `micro` gap.

---

## 4. Working with Promises and Async Operations

### Sequential vs parallel execution

```js
// sequential: each waits for the last → total ≈ sum of latencies
for (const id of ids) results.push(await fetchPost(id));
// parallel: all start now → total ≈ slowest latency
const results = await Promise.all(ids.map(fetchPost));
```

Sequential is simple and gentle on the far server but pays every latency one after
another. Parallel is fast but bursts — and, as §6 measures, one slow response inflates
the sequential total while parallel merely absorbs it. The middle ground is *bounded*
parallelism (that's `mapLimit`).

### `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`

| Combinator | Fulfills when | Rejects when | Ask it |
| --- | --- | --- | --- |
| `all` | **every** promise fulfills | **any** rejects (first one wins) | "did everything work?" |
| `allSettled` | always (array of `{status, value/reason}`) | never | "what happened to each?" |
| `race` | **first** to settle, either way | (its rejection, if first) | "who finishes first?" |
| `any` | **first** to fulfill | **all** reject (`AggregateError`) | "did anyone succeed?" |

Choosing in one breath: dependent results → `all`; partial failure is *data* →
`allSettled`; the *event* matters (timeouts!) → `race`; redundant fallbacks → `any`.
Subtlety: losers of a race are still "handled" (the combinator subscribed), but a promise
you create and never observe is an unhandled rejection waiting to happen.

### `Promise.withResolvers` and `Promise.try`

```js
// withResolvers: get the promise AND its resolve/reject handles up front
const { promise, resolve, reject } = Promise.withResolvers();
button.addEventListener('click', () => resolve('clicked'), { once: true });
cancelBtn.addEventListener('click', () => reject(new Error('cancelled')), { once: true });
```

Useful when the settle trigger lives *outside* the creating scope (events, cross-module
coordination) — the classic hand-rolled `{ resolve, reject }` pattern, now standard.

```js
// Promise.try: run a possibly-throwing function and get a promise — uniformly
await Promise.try(() => JSON.parse(input))   // rejects on SyntaxError...
  .catch(handle);                            // ...instead of needing try/catch around await
```

Its gift: synchronous throws become rejections, so mixed sync/async functions can be
handled in **one** place. (If your Node doesn't have it yet, `async () => fn()` is the
stand-in: it also converts sync throws to rejections.)

### Async iteration with `for await...of` and `Array.fromAsync`

```js
for await (const chunk of readableStream) { total += chunk.length; }
```

Same loop you know, but each `next()` may return a promise — the loop awaits it. Works
on async iterables: streams, cursors, and any object with `Symbol.asyncIterator`.

```js
const posts = await Array.fromAsync(ids, (id) => fetchPost(id));
// like Promise.all(ids.map(...)) but built for async mapping, in order
```

One line for "map every item asynchronously, collect in order" — the readable middle
ground between a `for` loop and `Promise.all(map(...))`.

**Choosing the right approach for a given problem.** Ask three questions: **Do I need
every result?** (no → `race`/`any`/first-hit loop; yes → next question) **Can partial
failure be tolerated or reported?** (`allSettled` vs `all`) **How much concurrency can
the target handle?** (everything → `all`; a cap → `mapLimit`; one-at-a-time →
`for…await`). When you're reaching for `forEach(async …)` or nested `await`s in loops,
you've outgrown the simple tools — that's the signpost back to this section.

> **Try it:** given `ids = [1..6]`, produce: (a) all results or throw on first failure,
> (b) a report of successes and failures, (c) whichever responds first. Write the three
> one-liners and say which combinator each used.

### Fetching and parsing JSON with `fetch`

```js
const response = await fetch('https://jsonplaceholder.typicode.com/posts/1');
const post = await response.json();      // await the body too — it's a promise
```

Two awaits, two failure points. `fetch` is built in (Node 18+, no package needed) and
returns a `Response` — headers/status are on the response, the body arrives via
`.json()`/`.text()`, which you must `await`.

`fetch` **rejects only on network-level failures** (DNS down, connection dropped). An
HTTP `404` or `500` is a *successful* fetch of an error page — you must check:

```js
if (!response.ok) {                      // ok = status 200–299
  throw new Error(`HTTP ${response.status} for ${url}`);
}
```

Forgetting `response.ok` is the classic "my try/catch never fires on a 404" bug.

`https://jsonplaceholder.typicode.com` needs no key, no auth, and returns predictable
data (`/posts`, `/todos`, `/users`, `?_limit=…`). It's free by design for exactly this
kind of exercise — so measured differences come from *your* code, not flaky credentials.
Still check `response.ok`: a rate-limit `429` arrives as a normal response.

> **Try it:** fetch `/posts/1` three ways — success; a bad URL (network error →
> rejection); a `404` path with `response.ok` unchecked vs checked. Watch which ones your
> `catch` actually sees.

### Timeouts and cancellation with `AbortController` and `AbortSignal.timeout`

```js
const controller = new AbortController();
setTimeout(() => controller.abort(), 3000);            // you decide when
const res = await fetch(url, { signal: controller.signal });
// later: controller.abort()  →  the fetch rejects with AbortError
```

One controller → one signal → pass `{ signal }` to every API that accepts one. The
modern one-liner for "give up after N ms": `AbortSignal.timeout(3000)` — a ready-made
signal that aborts itself. You can even combine them:
`AbortSignal.any([controller.signal, AbortSignal.timeout(3000)])` — whichever fires
first wins.

- **Timeout** = "this is taking too long" — a policy you set, expressed as
  `AbortSignal.timeout(ms)` (or `withTimeout(promise, ms)` when the target takes no
  signal).
- **Cancellation** = "I no longer want this" — user closed the dialog, navigated away;
  `controller.abort(reason)` is *your* call, possibly with a custom reason.

Both surface as a rejected promise with an `AbortError`-style reason — so **don't**
blindly `catch` and retry: a cancelled request retried forever is a bug. Check the
reason (`error.name === 'TimeoutError'` → retryable; `'AbortError'` from your own
controller → stop).

### Temporal: awareness only

`Date` is decades old: mutable-ish, easy to misuse across timezones, no duration type.
**Temporal** is the incoming standard replacement (`Temporal.Instant`, `Temporal.ZonedDateTime`,
`Temporal.Duration`) with a real API for arithmetic and timezones. No action needed here —
know it exists so `new Date()` arithmetic doesn't feel like the only possible world, and
check whether your Node ships it before using it anywhere.

---

## 5. Building Your Async Toolkit

Design first, then build each utility, then prove each one — every decision below has a
self-check that pins it down.

### Organizing the toolkit as an ES module

**Designing the toolkit's public API.** Decide **signatures and guarantees before
implementations** — the guarantees are what self-checks verify:

| Function | Contract |
| --- | --- |
| `sleep(ms)` | fulfills with `undefined` after ≥ `ms`; rejects `RangeError` on invalid `ms` |
| `retry(fn, {retries, delayMs})` | fulfills with `fn`'s value; after `retries + 1` attempts, **rethrows the last error**; rejects `RangeError` on bad options |
| `withTimeout(promise, ms)` | settles with the promise, or rejects `TimeoutError` (with `cause`) |
| `mapLimit(items, limit, fn)` | fulfills with results **in input order**; ≤ `limit` in flight; rejects as soon as any `fn` call rejects |

Design rules that pay off: **named exports** (a utility belt, not a default), **one
failure vocabulary** (every invalid input rejects — never sometimes-throw,
sometimes-reject), **never swallow errors** (rethrow the original, keep it as `cause`),
and **declare timing claims precisely** (`>= ms`, "at most limit") so assertions can be
literal.

```js
// async-toolkit.js
export class TimeoutError extends Error { /* ... */ }
export function sleep(ms) { /* ... */ }
export function retry(fn, opts) { /* ... */ }

// timing-comparison.js
import { sleep } from './async-toolkit.js';   // paths include the ./ and the extension
```

The chosen shape: one file, grouped in reading order — **errors first** (`TimeoutError`,
the vocabulary everything else speaks), **utilities** (each with its JSDoc: signature,
what it rejects with, what it guarantees), **self-checks** below a clear divider, and the
**entry-point guard** last. Public surface = `export`. Anything helper-ish stays
unexported. This keeps the deliverable ("one ES module") honest while still separating
*definitions* from *verification*.

### Implementing `sleep`

The teaching example — one `setTimeout` in a promise:

```js
export async function sleep(ms) {
  if (typeof ms !== 'number' || Number.isNaN(ms) || ms < 0) {
    throw new RangeError(`sleep(ms): ms must be a non-negative number, got ${ms}`);
  }
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

Why `async`? So the `RangeError` arrives as a **rejection** like every other utility's
bad input — a function that throws sync for bad arguments but rejects for real failures
forces callers to handle two shapes. And why validate at all? `setTimeout` happily
accepts `-1` (treats it as 0) and `'50'` (coerces) — silently plausible answers to wrong
questions.

**Self-check shape:** measure `Date.now()` before/after — assert it waited `>= ms` — plus
`assert.rejects(sleep(-1), RangeError)`.

### Implementing `retry(fn, { retries, delayMs })`

Skeleton to build against (fill in the loop yourself):

```js
export async function retry(fn, { retries = 3, delayMs = 100 } = {}) {
  // 1. validate: retries is a non-negative integer, delayMs is >= 0 → else RangeError
  // 2. loop attempt = 1 .. retries + 1:
  //      try   { return await fn(attempt) }        ← pass the attempt number so fn can log
  //      catch { remember the error; if attempts remain → await sleep(delayMs) }
  // 3. loop ended without success → throw the LAST error (never a generic "failed")
}
```

Decisions to make and defend: what does `retries: 0` mean (exactly one attempt — nothing
retried), why pass `attempt` into `fn`, why the *last* error (callers need the real
cause), and why validate options loudly (`retries: -1` silently returning on first try is
a lie the self-check can't catch later).

**Self-check shape:** a function that fails twice then succeeds (`retries: 3` → value
after exactly 3 attempts); one that always fails (assert the *message of the last
attempt*, not just "it threw"); timing: two retries at `delayMs: 50` must take
**≥ 100ms** — proving the delay exists, not just the loop.

### Implementing `withTimeout(promise, ms)`

The shape: **race** the real promise against a timeout that rejects.

```js
export function withTimeout(promise, ms) {
  const signal = AbortSignal.timeout(ms);          // validates ms for you (RangeError)
  const timeout = new Promise((_, reject) => {
    signal.addEventListener('abort', () =>
      reject(new TimeoutError(ms, { cause: signal.reason })), { once: true });
  });
  return Promise.race([promise, timeout]);         // first to settle wins
}
```

Points to internalize: `TimeoutError extends Error` carries `cause` = the abort reason
(§2's context rule, in code); `Promise.race` **subscribes to both branches**, so neither
the timeout nor a late real rejection becomes an *unhandled* rejection; `{ once: true }`
keeps listeners tidy; and `withTimeout` doesn't cancel the original promise — it stops
*waiting*. True cancellation needs the promise's API to accept a signal (as `fetch` does).

**Self-check shape:** (a) fast promise passes through with its value; (b) slow promise →
`TimeoutError`, `error.cause` present, and the rejection arrives *before* the slow
promise would have finished (assert elapsed `< slowMs`); (c) a real rejection comes
through as itself — `withTimeout(Promise.reject(new Error('boom')), 500)` must reject
`/boom/`, not time out.

### Implementing `mapLimit(items, limit, fn)`

The hardest of the four — build it in this order:

1. **Naive first:** `for (const item of items) out.push(await fn(item))` — correct but
   sequential. Name the bug: total time = sum of latencies.
2. **Shared state:** a pre-sized `results` array (input order survives out-of-order
   completion — assign by index, don't `push`) and one shared `nextIndex` counter
   (`nextIndex++` is safe — JavaScript is single-threaded; no locks needed).
3. **Workers:** an `async worker()` that loops — take the next index, bail if none left,
   `results[index] = await fn(items[index], index)`, repeat.
4. **Pool:** spawn `Math.min(limit, items.length)` workers, `await Promise.all(workers)`,
   return `results`.
5. **Validate** `limit` is an integer `>= 1` (a `0` would spawn no workers and return
   holes — silence, not an error).

The subtlety worth an hour of thought: `Promise.all(workers)` rejects fast when a `fn`
call rejects — in-flight siblings still finish quietly, but because `Promise.all`
subscribed to *every* worker, none of their outcomes can drift into an
unhandled-rejection warning. Fail fast, swallow nothing.

**Self-check shape:** order (`deepEqual` vs `items.map(...)`), a **concurrency meter**
(counters inside the test's own `fn`: assert `maxActive <= limit` *and*
`maxActive === limit` — the cap and the parallelism both proven), `limit: 1` behaves
sequential, empty list → `[]`, one throwing item rejects the call, `limit: 0` →
`RangeError`.

### Writing an individual `node:assert` self-check for each utility

Start with the assertion library itself:

```js
import assert from 'node:assert/strict';

assert.equal(await sleep(30), undefined);          // throws AssertionError on mismatch
await assert.rejects(mapLimit([1], 0, fn), RangeError);  // expected-rejection form
```

`assert` **throws** on failure — run the file with `node` and a failure is a non-zero
exit code, so it works as a gate in scripts and CI. Use `node:assert/strict` (strict
equality everywhere; the loose variant quietly allows `1 == '1'`). Assertions are your
earliest "tests" — M09 will wrap this instinct in a real framework.

"Independent" means each utility's checks stand alone: no check depends on another
utility's correctness *except* where it's the explicit subject (e.g., `retry`'s delay
check uses `sleep`). Structure: `import assert from 'node:assert/strict'`, one `check
name, async fn` harness that reports `ok - name` / `FAIL - name` and counts at the end,
and **run the checks only when the file is the entry point**:

```js
const entry = process.argv[1] ? path.resolve(process.argv[1]) : null;
const isEntryPoint = import.meta.main ?? (entry === fileURLToPath(import.meta.url));
if (isEntryPoint) await runSelfChecks();
```

Otherwise importing `sleep` into the timing script would silently re-run all checks (or
worse — if you got the guard backwards, your "passing" output would never run at all).

### Handling edge cases and failures

Run every utility past its edges before trusting it: `0`/empty/negative/one-item inputs,
a `fn` that throws immediately vs halfway, `limit > items.length`, an already-rejected
promise into `withTimeout`, options objects with missing/extra/negative fields. For each,
ask: *does it fail loudly (RangeError/TimeoutError) or answer quietly?* — quiet wrong
answers are the bugs that survive review. Real trade-offs you're accepting: fixed delay
vs exponential backoff in `retry`; fail-fast vs fail-soft in `mapLimit`; timeout that
*stops waiting* vs true cancellation in `withTimeout`. Write down which one you chose and
why — that sentence is write-up material.

> **Try it:** before running anything, write the four self-check names for one utility on
> paper. If you can't name a check, you haven't decided what "correct" means yet — fix
> that first, then code.

---

## 6. Measuring Sequential and Parallel Requests

### Fetching JSONPlaceholder data sequentially

```js
const posts = [];
for (const id of POST_IDS) {
  posts.push(await fetchPost(id));   // clock is running the whole time
}
```

Every request starts only after the previous one **fully** finished — total time ≈ sum
of all latencies, including the slowest one at full price.

### Fetching the same data in parallel

```js
const posts = await Promise.all(POST_IDS.map((id) => fetchPost(id)));
```

All requests are initiated in the same tick; `Promise.all` just waits for the results —
total time ≈ the *slowest single* request (everything else finished inside that window).

### Measuring and recording actual execution times

```js
const started = Date.now();
await run();
console.log(`sequential ${Date.now() - started}ms`);
```

Measurement hygiene — each of these changed a number during this module:

- **Warm up first:** one throwaway request before the clock, or request #1 pays DNS + TLS
  and poisons whichever run goes first.
- **Same inputs both ways** (same ids, same order of *results*) or you're comparing
  different work.
- **Run more than once** and record every run: network latency varies run to run, so a
  single number is an anecdote. Expect variance, not a fixed ratio.

### Explaining the observed timing difference

Sequential waits for the *round trip* each time: if each of 8 requests takes ~100ms of
real network time, sequential pays 8 × 100ms — while parallel pays ~100ms total, because
the waiting happens simultaneously. Nothing got faster; the waiting **overlapped**. The
cleanest proof is a single slow response: it inflates the sequential total by its full
latency (measured runs here: a 2236ms sequential outlier) while the parallel run merely
*includes* that latency inside its overlap (same data, 39ms).

### Understanding when concurrency limits are useful

Parallelism helps when waits dominate (network, disk) and the target tolerates concurrency
— it does nothing for pure CPU work (one thread!), and it can *hurt* when it triggers rate
limits, exhausts connection pools, or makes debugging a thundering herd. Real-world
default: **bounded** parallelism — pick a limit from what the service documents (or start
low), then measure.

**Connecting concurrency limits to `mapLimit`.** `Promise.all(map(...))` is "limit = N,
no negotiation". `mapLimit(items, 8, fn)` is the tunable version of the same measurement
— and the reason it exists: fetch the same data *without* the burst. Worth measuring as a
third run: `sequential`, `Promise.all`, and `mapLimit(posts, 3, fetchPost)` — expect the
middle to win on time, the first to be safest for the server, and `mapLimit` to sit
between them with a knob. If `mapLimit(..., limit)` ever returns *slower* than
sequential, your limit is effectively 1 — check the meter in the self-check before
blaming the network.

> **Try it:** add a third measurement to the timing script using your `mapLimit` with
> `limit: 3`, run everything three times, and paste all nine numbers into your write-up.

---

## 7. Refactoring Callback-Based Code

### Understanding callback-style `node:fs` code

The pre-promise `node:fs` API is error-first callbacks: every operation takes
`(args..., (err, result) => {})`. Chaining dependent operations means nesting, because
each step needs the previous step's output — and each level needs its own error branch:

```js
fs.writeFile(input, data, (err) => {
  if (err) { report(err); return; }
  fs.readFile(input, 'utf8', (err, raw) => {
    if (err) { report(err); return; }
    fs.writeFile(output, tidy(raw), (err) => {
      if (err) { report(err); return; }
      // ...
    });
  });
});
```

Four operations, four nesting levels, four `if (err)` blocks that must each be *individually*
correct.

### Refactoring to promises and `async`/`await`

Node ships both worlds: `node:fs` (callbacks) and `node:fs/promises` (promises) — same
operations, same names, different shape:

```js
import fs from 'node:fs/promises';   // writeFile, readFile, mkdir, ...
```

The mechanical conversion: drop the callback, `await` the returned promise, delete every
`if (err)` — the rejection *is* the error branch now. Before/after must be **behaviorally
identical** (same output, same failure outcomes): refactor is not a rewrite, so prove
equivalence by running both and diffing the results.

**Rewriting the flow with `async`/`await`:**

```js
async function main() {
  await fs.mkdir(dir, { recursive: true });
  await fs.writeFile(input, NOTES);
  const raw = await fs.readFile(input, 'utf8');
  await fs.writeFile(output, tidy(raw));
  const result = await fs.readFile(output, 'utf8');
  console.log(result);
}
```

The pyramid becomes a **sequence**: steps in order, indentation flat, variables from step
2 still in scope at step 5 (no closures over deep levels). The only structural addition
is **one** `try/catch` around the whole flow (or around just the steps whose failure you
handle differently).

### Handling errors correctly

The refactor's failure mode is subtler than the callback's: a missing `await` or a
bare `main()` call turns every error into an unobserved rejection. Rules:

- **One `catch` for the flow** — log `error.message` (or a labeled prefix per step) and
  set `process.exitCode = 1`; a caught-and-ignored error is the same bug as an empty
  `catch`.
- **Rethrow or translate with `{ cause: error }`** when you catch mid-flow and rethrow —
  losing the original is how "ENOENT" becomes "something went wrong".
- **Never `catch {}`** in a refactor — you've traded visible errors for invisible ones.
- If `main()` is called, make its rejection handled: `try { await main(); } catch` at the
  top level (top-level `await` inside `try` works in modules).

### Comparing the original and refactored code

Walk both files side by side and judge them on: **error handling** (N hand-written
branches vs one `catch`), **scope** (nested closures vs top-to-bottom locals), **failure
detection** (an `if (err)` you might forget vs an exception that forces the issue),
**control flow** (each step's continuation lives inside the previous callback — reasoning
requires the *nesting*, not the reading order — vs plain sequential reading order), and
**composability** (can't wrap steps in `try`/loops vs loops and `try` work normally). The
point isn't "callbacks are bad" — it's that promises moved the error bookkeeping *into
the language*, so your code can spend its indentation on structure instead.

> **Try it:** delete one `if (err)` branch from the callback version and one `await` from
> the async version, run both, and observe which failure is louder. The difference in
> "loudness" is the whole argument.

---

## 8. Avoiding Common Async Mistakes

Read this section *after* your first build attempt — every item below is a bug you're now
positioned to recognize in your own code.

### Forgetting `await`

```js
const data = readFile(path);      // data is a Promise!  .toString() → "[object Promise]"
console.log(data.length);         // undefined — no error, just wrong
```

Symptom: `undefined`, `[object Promise]`, or "cannot read property X of undefined"
where a value clearly exists. Fix: `await` it. Discipline: if a function returns a
promise, treat *every* use of its result as needing an await unless you deliberately
fire-and-forget (and then `.catch(...)` explicitly).

### Using `await` inside loops unnecessarily

```js
for (const id of ids) results.push(await fetchPost(id));   // correct... and 8× slower
```

Two distinct problems wearing one coat: (1) **slowness** — sequential waits multiply;
only keep it when you *need* the order or the next iteration depends on the previous
result (retries, dependent writes). (2) **Accidental serialization** of things that
could overlap. Reach for `Promise.all(map(...))` or `mapLimit` instead — and never
`await` where the loop variable itself isn't used by the await's result chain.

### Using `forEach(async ...)`

```js
ids.forEach(async (id) => { await save(id); });   // ❌ returns undefined immediately
await doTheThing();                                 // runs BEFORE saves finish
```

`forEach` ignores your callback's return value — no promise to await, no error to catch,
work flies loose. Fixes, in order of preference: `for…of` + `await` (clearest),
`Promise.all(ids.map(async (id) => ...))` (parallel), `mapLimit` (bounded). Rule:
**`forEach` never appears in code that contains `await` inside its callback.**

### Unhandled promise rejections and swallowed errors

A rejection with no handler anywhere: starts as a Node warning, and in modern Node an
`unhandledRejection` **crashes the process** — often minutes after the real bug, pointing
at the wrong place. Typical sources: the `forEach` above, a floating `someAsyncFn()`
without `await`/`.catch`, or a rejected promise nobody raced/collected. Find them: run
with `node --unhandled-rejections=strict` (turns them into immediate crashes with stacks)
while testing; add `process.on('unhandledRejection', log)` during development for a
friendlier net.

```js
try { await risky(); } catch { }                    // swallowed whole
await risky().catch(() => {});                      // same bug, shorter
const v = await risky().catch(() => null);          // often fine — IF null is a real answer
```

An empty `catch` converts a failure into "nothing happened" — downstream code then fails
*far* from the cause with a misleading message. If you *choose* a fallback, make it a
value that's correct for every caller (and say so in a comment); if you don't know what
to do, rethrow — with `{ cause }` if you add context. Also: catching only
`error.message` (string matching) instead of custom error types (§2) is *incorrect error
handling* dressed as convenience.

### Unexpected execution order and race conditions

Classic shapes: starting two async steps that write the same state (last-to-finish wins,
whichever that is); `Promise.race` where you actually needed `Promise.all`; aborting a
request but ignoring the `AbortError` and retrying anyway; a floating promise from step 1
still running when step 2 reads what step 1 was supposed to write. Tell-tale symptom:
**failures that only appear sometimes**, or ordering that changes when you add
`console.log` (Heisenbug — your logging changed the microtask timing). Fixes: await the
thing you depend on, make writes single-flight (one `await`ed code path per resource),
and treat "it works locally, flaky in CI" as a queue-order question until proven
otherwise.

### Troubleshooting failed self-checks

- **Read the assertion message first** — `expected >= 100ms, took 12ms` tells you the
  *delay never ran*, not that your stopwatch is wrong (that's a `retry` bug, not a timing
  bug).
- **`14/15` with a "failure" whose message is an unrelated `RangeError`** means the check
  *threw before it asserted* — the code path the check exercised is inconsistent (this
  exact shape caught `sleep` throwing synchronously while its siblings rejected).
- **`expected maxActive === 3, got 1`** — count workers: `Math.min(limit, 0)`?
  Actually `items.length` 0? Or you awaited inside the pool-creation, serializing it.
- **Timing runs that disagree wildly** — check warm-up, connection reuse, and whether the
  API itself is having a moment; report medians of 3–5 runs, not the first number.
- **Works with `node file.js`, fails on import** — your entry-point guard is inverted;
  importing must not *run* checks, and running directly must.

> **Try it (the module's diagnostic rite of passage):** take any snippet from §3, predict
> its order on paper, run it, and if you're wrong — find the exact line where your mental
> queue model diverged from the real one. That line is your personal pitfall list.

---

## 9. Hands-On Practice

Work these in order — each one earns the next. Sections 2–4 were your research: if those
`tasks.md` boxes aren't ticked yet, work them with a scratch file first. Use the workflow
from §1 — research before (or alongside) the build, verify with something executable,
document while it's fresh.

1. **Build `sleep`, `retry`, `withTimeout`, and `mapLimit`, each with its own passing
   `node:assert` self-check.**
   `sleep` → `retry` → `withTimeout` → `mapLimit`, each with its own `node:assert`
   checks (shapes in §5). Run `node async-toolkit.js` — every check must pass, exit
   code 0, no unhandled-rejection warnings.
2. **Fetch JSONPlaceholder data sequentially and in parallel, then record and compare the
   actual timings.**
   Same ids both ways, warm-up fetch first, `Date.now()` around each (§6). Record
   *every* run's numbers — they go in your write-up's checkpoint evidence, and parallel
   must win.
3. **Refactor callback-style `node:fs` code to promises and `async`/`await`.**
   Write the callback version first (or take the pyramid in §7), convert to
   `node:fs/promises`, prove identical output, and one deliberate missing-`await` bug to
   feel the failure.
4. **Predict and explain the output order of a snippet mixing `setTimeout`, promises, and
   `await`.**
   Write the prediction *before* running (§3 procedure), run it, then explain each
   position — especially the `await`-continuation vs chained-`.then` pair.
5. **Review the toolkit, verify all self-checks, and confirm the required outcomes.**
   Re-read your toolkit against every Definition of Done checkbox in `brief.md`. Ask:
   does any check pass for the *wrong reason*? Do all invalid inputs fail loudly? Capture
   verification evidence as you go — self-check output, every timing run, the
   prediction/actual transcript (keep evidence files local and gitignored — never
   committed) — and fill `write-up.md` section by section: what you built, key decisions,
   one teach-it guide, concepts in your own words, what tripped you up, checkpoint
   evidence, what you'd do differently.
6. **Continue using your existing `node-js-training-ground` repository. Create an M02
   feature branch, commit your work, open a pull request, and merge it.**
   Conventional Commits on the feature branch, push, then open the PR (describe what you
   built and what you measured), and merge. The PR history *is* your public checkpoint
   trail.

---

## 10. Wrap-Up and Self-Assessment

Two passes: prove your understanding with the knowledge check (answer without looking
back, then check the key), then confirm your deliverable with the self-assessment that
follows. For each area, the honest test is: *could you explain it to the next trainee
with a fresh example, not the one in the text?*

### Knowledge check

**Modern JavaScript syntax and data manipulation**

1. Why does `credits || 3` misbehave for a user with `credits: 0`, and what's the fix?
2. `map` vs `flatMap` — when does each return an array of the same length as its input?
3. Name two advantages a `Map` has over a plain object as a cache.
4. What does `structuredClone` copy that `{ ...obj }` does not?

**Modules, classes, and error handling**

5. Your toolkit exports four functions — named or default exports, and why?
6. Why does `class TimeoutError extends Error` need `this.name = 'TimeoutError'`?
7. You catch an error and throw a new one. What one option keeps the original failure
   visible in logs?
8. What does the entry-point guard around `runSelfChecks()` prevent?

**Event loop, tasks, microtasks, and output ordering**

9. Order check — what prints, and why?
   ```js
   console.log(1);
   Promise.resolve().then(() => console.log(2));
   setTimeout(() => console.log(3), 0);
   console.log(4);
   ```
10. Why can `await null`'s continuation beat a chained `.then` into the queue?
11. True or false: `setTimeout(fn, 0)` runs "immediately after the current line". Why?

**Promise methods and async/await error handling**

12. Match to a combinator: "the first one to finish wins, whatever it was" / "I need every
    result or a failure" / "give me a report of successes and failures" / "any success
    will do".
13. Why is `async function f() { try { await g(); } catch {} }` a trap when `f`'s caller
    expects to know about failures?
14. What's wrong: `posts.forEach(async (p) => { await save(p); }); await done();`?

**Sequential, parallel, and concurrency limits**

15. Sequential = 8 requests × ~100ms. What's the approximate parallel total, and what
    does that number actually represent?
16. Why did one 2236ms run inflate the sequential total but not the parallel one?
17. You need parallel speed but the API allows 5 concurrent requests. Which tool and why?

**Timeouts, cancellation, and common async pitfalls**

18. `AbortSignal.timeout(3000)` vs `controller.abort()` — which is a policy, which is a
    decision, and what do both surface as?
19. Why does `fetch` reject on a dropped connection but *not* on HTTP 500?
20. Your self-check reports `14/15` and the failure message is a `RangeError` from code
    the check never asserted on. What's the likely story?

**Can each toolkit utility be explained and verified independently?** For each of
`sleep`, `retry`, `withTimeout`, `mapLimit`: state its contract in one sentence, its
failure mode for bad input, and name two self-checks that pin it down. If you can't do
all three for all four, revisit §5 before calling the module done.

#### Answer key

1. `0` is falsy, so `||` substitutes `3` — use `??`, which only falls through on
   `null`/`undefined`.
2. `map` transforms one-to-one (same length always); `flatMap` maps one-to-many and
   flattens one level — its length follows the inner arrays.
3. Any-key keys (functions/objects, not just strings), `O(1)` `has` lookups, plus
   `.size` and guaranteed insertion order. (Objects still win for JSON-shaped data.)
4. `structuredClone` recurses into nested objects/arrays and makes genuinely independent
   copies; spread copies only one level, so nested references are still shared. It also
   clones `Map`/`Set`/`Date`/buffers — and throws on functions/class instances, where
   spread silently keeps the same reference.
5. Named exports: explicit imports, no "what does this file export?" guesswork, better
   for tooling; a belt of utilities has no single main item.
6. Class names are not painted onto `error.name` automatically — and logs/`instanceof`
   checks read on `name`/`class`, not on the string you passed to `super`.
7. `{ cause: error }` (second constructor argument) — the outer error explains the task,
   the cause keeps the root failure.
8. Importing the module must not *run* checks (side-effect-free imports — the timing
   script imports `sleep`), while running it directly must (that's the gate).
9. `1, 4, 2, 3` — sync prints 1 and 4; the `.then` is a microtask, drained before the
   event loop takes the `setTimeout` task.
10. Because the `await` continuation is queued *when the await executes* (during the
    sync pass), while the chained `.then` doesn't even exist until its predecessor's
    handler runs later in the microtask drain.
11. False. It's the next trip around the loop — every already-queued microtask drains
    first, which is why `3` lands after `2`.
12. `race` / `all` / `allSettled` / `any`, in that order.
13. An empty `catch` swallows the rejection — the caller's `await f()` succeeds with
    `undefined` and no one downstream learns the truth. Rethrow (with `cause`) or return
    a *deliberate*, documented fallback.
14. `forEach` ignores the async callbacks' promises: `done()` runs before any `save`
    finishes, and any save-rejection is unhandled. Use `for…of` + `await` or
    `Promise.all(map(...))`.
15. ~100ms — the *slowest single* request, not the sum: the waiting overlaps. It
    represents wall-clock latency of one round trip, not total work done.
16. Sequential waits for *every* response in turn, so the outlier adds its full latency
    to the total; parallel was already waiting ~that long anyway, so the outlier lands
    inside the overlap — the total barely moves.
17. `mapLimit(items, 5, fn)` — bounded parallelism: `Promise.all` would burst all of
    them, sequential gives up the speed; the limit is the knob.
18. Timeout = time policy (`AbortSignal.timeout`); cancellation = your decision
    (`controller.abort()`); both surface as a rejection with an abort-style reason —
    check the reason before blindly retrying.
19. Network-level failure (DNS/connection) rejects the promise; HTTP errors are a
    *successful* response with a bad status — check `response.ok` yourself.
20. The check threw (or hit an assert) for a *different* reason than the behavior it
    meant to pin down — usually validation that throws synchronously where the check
    expected a rejection (or vice versa). Fix the production inconsistency, not the
    message.

### Explain the event loop and asynchronous output ordering

**Queues, not magic.** Sync code → drain microtasks → one task. If you can narrate that
loop, output ordering and most flakiness stop being mysterious. Re-run the §3 prediction
snippet and narrate each queue state out loud until the order is obvious, not memorized —
that narration *is* the checkpoint.

### Explain the design choices behind your toolkit

**Make failure loud and uniform.** Every bad input rejects the same way; every real
error keeps its original (`cause`); nothing is caught-and-ignored. Your toolkit's value
is its *predictable failure*, not its happy path. **Pick concurrency deliberately.**
`Promise.all` (all at once), `mapLimit` (a knob), sequential (a choice) — three answers,
three different costs to the server and to you. Be ready to defend the trade-offs you
accepted: fixed delay vs exponential backoff in `retry`, fail-fast vs fail-soft in
`mapLimit`, stop-waiting vs true cancellation in `withTimeout`.

### Confirm all self-checks pass

**Review of the completed Async toolkit and captured evidence.** Re-walk the deliverable:
four utilities in one ES module, **15/15 self-checks passing** (exit 0, no warnings);
three timing runs where parallel won every time (2236→39ms, 132→37ms, 100→36ms);
both fs-refactor versions producing identical output; the output-order prediction
matching the actual run position for position; evidence files kept local and gitignored;
and the checkpoint PR merged in the training-ground repo. Each DoD checkbox should point
at a *thing you can show*, not a memory.

**Final Definition of Done checklist**

- [ ] All `node:assert` self-checks pass (`node async-toolkit.js` → exit 0).
- [ ] You can predict and explain, unaided, the output order of a snippet mixing
      `setTimeout`, promises, and `await`.
- [ ] The timing comparison shows parallel beating sequential, and you can explain why.
- [ ] `tasks.md` fully ticked; `write-up.md` complete with real evidence; branch → PR →
      merged for the lab checkpoint.

### Explain the results of your sequential vs parallel timing comparison

**Measure instead of assuming.** Parallel *should* win — you still ran it three times,
warmed up first, and recorded all the numbers. The 2236ms run only became a lesson
because it was written down: sequential *sums* latencies (one slow round trip inflates
the whole loop), parallel *overlaps* them (the same outlier lands inside the overlap).
Walk your own table and say both halves out loud — what the numbers are and why they
have to be that way.

### Identify and correct common asynchronous programming mistakes

- `forEach(async …)`, floating promises, and `await`-less promises used as values.
- Empty `catch` blocks and swallowed rejections — errors that used to be bugs become
  *invisible* bugs.
- `response.ok` never checked; `||` used where `??` was meant; string-matching on error
  messages instead of error types.
- One giant `Promise.all` where the API documents a concurrency limit.
- Trusting the first timing number (or the config file that says which Node is "active")
  without asking the tool.

### Check your readiness to apply modern JavaScript and async patterns in the next module

M03 moves from *language-level* async to the *Node runtime*: core modules beyond
`node:fs`, npm and `package.json`, and a bare `node:http` server — all of which you'll
write **asynchronously**. Bring: the queue narration (you'll need it when a server
callback fires "unexpectedly"), the error discipline (an HTTP handler that swallows a
rejection takes a request down silently), the `mapLimit` mindset (servers throttle
concurrency for a living), and the branch → PR → evidence habit — M03's brief will be
the next folder in both repos, and the same research → build → verify → document loop
applies from line one.
