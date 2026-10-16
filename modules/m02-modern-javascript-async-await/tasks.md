# M02 Tasks — Modern JavaScript & Async/Await

Write idiomatic modern JavaScript, explain the event loop at a conceptual level, and write reliable
asynchronous code.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully (and skim your m01 write-up) before starting.
- [x] Continue the branch → PR → merge habit from M01 for this lab's checkpoint.

## Research (do before or alongside the build, not after)

- [x] `const`/`let`, scope, closures, arrow functions & `this`, template literals.
- [x] Destructuring, spread/rest, default parameters, optional chaining `?.`, nullish coalescing
      `??`/`??=`, and the truthy/falsy/equality pitfalls.
- [x] `map`/`filter`/`reduce`/`find`/`some`/`every`/`flatMap`, the non-mutating helpers
      `toSorted`/`toReversed`/`with`, `structuredClone`, and `Object.groupBy`/`Map.groupBy`.
- [x] `Map` and `Set` (including the newer `Set` methods), classes with `#private` fields,
      getters/setters.
- [x] `try/catch/finally`, writing custom error classes, and the `Error` `cause` option.
- [x] ES module named vs default exports — decide which you'll use in the toolkit.
- [x] The call stack, the event loop, and the task-queue vs microtask-queue distinction — you'll
      need this to explain output ordering later.
- [x] Callbacks → Promises (`then`/`catch`/`finally`, chaining) → `async`/`await`, and how error
      handling differs in async code.
- [x] `Promise.all`, `allSettled`, `race`, and `any` — and when each fits.
- [x] `Promise.withResolvers`, `Promise.try`, and async iteration (`for await…of`,
      `Array.fromAsync`).
- [x] `AbortController` and `AbortSignal.timeout` for cancellation and timeouts.
- [x] `fetch` and working with JSON responses.
- [x] The common async pitfalls: forgotten `await`, `await` inside loops, `forEach(async …)`,
      unhandled rejections, swallowed errors — know what each looks like before you build.
- [x] Note (awareness only): Temporal exists as the future replacement for `Date` — you don't need
      to use it here.

## Build

- [x] Build `sleep`, with its own `node:assert` self-check.
- [x] Build `retry(fn, { retries, delayMs })`, with its own `node:assert` self-check.
- [x] Build `withTimeout(promise, ms)`, with its own `node:assert` self-check, using what you
      researched on `AbortController`/`timeouts`.
- [x] Build `mapLimit(items, limit, fn)`, with its own `node:assert` self-check.
- [x] Fetch from JSONPlaceholder sequentially, then in parallel, and compare the timings.
- [x] Refactor a piece of callback-style `node:fs` code to promises/async-await.

## Verify

- [x] All `node:assert` self-checks pass.
- [x] You can predict and explain, unaided, the output order of a `setTimeout`/promise/`await`
      snippet.
- [x] The timing comparison shows parallel beating sequential, and you can explain why.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built": log each toolkit function as you finish it, not all at the end.
- [x] "Why it's built this way (key decisions)": how you implemented `retry`/`mapLimit`, what
      would change if you'd fetched the other way (sequential vs parallel), and which async
      pitfall you specifically designed around.
- [x] "How to build it (teach it to the next trainee)": write the guide for one utility of your
      choice.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": note forgotten-`await` or race-condition surprises as they happen.
- [x] "Checkpoint evidence": your self-checks passing, your actual timing numbers, and your
      explanation of the `setTimeout`/promise/`await` output order.
- [x] Close out "What I'd do differently".
