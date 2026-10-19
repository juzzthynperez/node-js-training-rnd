# M02 Write-Up — Modern JavaScript & Async/Await

**Trainer:** juzzthynperez · **Date:** 2026-10-16 · **Repo:** [juzzthynperez/node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground) (public) · **Merged PR:** [#3 — feat(m02): async toolkit, timing comparison, and fs refactor examples](https://github.com/juzzthynperez/node-js-training-ground/pull/3)

## What I built

The **Async toolkit** ES module plus the two lab companions, all under
`modules/m02-modern-javascript-async-await/` in the training-ground repo:

- **`async-toolkit.js`** — four utilities, named exports, one module:
  - `sleep(ms)` — a `setTimeout` wrapped in a promise. The building block the other
    self-checks use for their timing assertions.
  - `retry(fn, { retries, delayMs })` — attempts `fn` up to `retries + 1` times, waits
    `delayMs` between attempts, passes the 1-based attempt number into `fn` so it can log
    what it is doing, and **rethrows the last error** when attempts run out.
  - `withTimeout(promise, ms)` — races the promise against `AbortSignal.timeout(ms)` and
    rejects with a custom `TimeoutError` (with `cause` carrying the abort reason) if the
    promise is too slow.
  - `mapLimit(items, limit, fn)` — maps with at most `limit` calls in flight, using a fixed
    pool of workers sharing one index counter; results come back in input order.
  - Self-checks with `node:assert` live in the same file as `runSelfChecks()` and only run
    when the file is the entry point, so importing the toolkit has no side effects.
    **Final run: 15/15 passed.**
- **`timing-comparison.js`** — fetches the same 8 JSONPlaceholder posts sequentially, then
  in parallel with `Promise.all`, and prints both wall-clock times (with a throwaway
  warm-up fetch so DNS/TLS setup doesn't skew either measurement).
- **`fs-refactor/callback-before.js` and `fs-refactor/async-after.js`** — the same
  write → read → transform → write job first as a four-level callback pyramid with an
  `if (err)` at every level, then flat with `node:fs/promises` and one `try/catch`. Both
  print identical output.

Three commits, merged as PR #3:
`feat(m02): add async toolkit with node:assert self-checks` →
`feat(m02): add sequential vs parallel timing comparison` →
`feat(m02): add callback-to-async fs refactor examples`.

## Why it's built this way (key decisions)

- **`retry` uses a fixed `delayMs`, not exponential backoff.** Backoff is the production
  answer, but it hides what `retry` actually does behind a formula. A fixed delay makes the
  self-check honest — two retries at `delayMs: 50` must take **≥ 100ms**, and the assertion
  proves the delay really happens instead of being skipped. The last error is rethrown
  rather than a generic one, so callers see which attempt actually failed.
- **`mapLimit` is a worker pool with a shared index, not "chunk the array."** Chunking
  (`[items.slice(0, limit), ...]`) runs each chunk in lockstep — one slow item stalls its
  whole chunk. Workers pulling the next index keep `limit` slots busy: whichever worker
  finishes first takes the next item. Results go into a pre-sized array at the original
  index, so input order survives even though completion order doesn't. It rejects as soon as
  any call rejects (fail fast) — in-flight siblings finish quietly, but because `Promise.all`
  subscribed to every worker, no rejection can drift into an unhandled-rejection warning.
  That trade-off (fail fast, keep it simple) is deliberate for a first pass at concurrency.
- **`withTimeout` builds on `AbortSignal.timeout(ms)` rather than a hand-rolled
  `setTimeout` reject.** The brief makes `AbortController`/`AbortSignal.timeout` required
  learning, and expressing "give up after N ms" as an abort signal is the same vocabulary
  you later use to cancel a `fetch`. `Promise.race` subscribes to both branches, so neither
  the timeout nor the real rejection can become unhandled — a rejected promise that loses
  the race is still *handled*.
- **Sequential vs parallel fetch — what would have changed the other way.** Sequential took
  **2236ms / 132ms / 100ms** across three runs; parallel took **39ms / 37ms / 36ms**.
  Sequential multiplies latency by N (each request waits for the previous one's full
  round-trip) and it makes every request's variance add up — run 1 shows that: one slow
  request ballooned the whole loop to 2236ms, while the parallel run absorbed that same
  latency inside its wall-clock overlap and still finished in 39ms. The flip side: parallel
  fires all N requests at once, which is exactly what gets you rate-limited on a real API —
  which is why `mapLimit` (bounded parallelism) is the tool you reach for in production, not
  `Promise.all` on everything.
- **The pitfall I specifically designed around: `forEach(async …)` and forgotten `await`.**
  `forEach` ignores the promises its callback returns — errors vanish and nothing waits.
  `mapLimit` exists because it *does* wait: it awaits inside its worker loop and aggregates
  workers with `Promise.all`, so a rejected call reaches the caller. The self-check
  `rejects when any call rejects` is the executable version of that design rule, and the
  entry-point guard on `runSelfChecks()` prevents the other classic — accidentally running
  side effects (a test suite, a fetch) just because a file was imported.

## How to build it (teach it to the next trainee)

**Guide: building `mapLimit`, the concurrency-limited map.**

1. **Start from the naive version and name the bug.** The obvious implementation is
   `for (const item of items) results.push(await fn(item))` — that's sequential: correct,
   order-safe, but every item pays the full wait. The second obvious version is
   `Promise.all(items.map(fn))` — parallel, but with no ceiling; 1000 items means 1000
   simultaneous calls. The exercise is the *middle*: at most `limit` in flight, results in
   input order.

2. **Decide the shared state before writing code.** You need (a) a place for results that
   survives out-of-order completion and (b) a way to hand out work. Results: a pre-sized
   `new Array(items.length)` so `results[index] = …` keeps input order no matter when each
   call finishes. Work hand-out: one shared `nextIndex` counter that workers increment —
   single-threaded JavaScript makes `nextIndex++` safe without locks.

3. **Write one worker as a `while` loop.** A worker is just: grab the next index, stop if
   there is nothing left, `await fn(items[index], index)`, store the result, repeat. The
   loop is what makes workers keep pulling work instead of handling one item and dying.

   ```js
   async function worker() {
     while (nextIndex < items.length) {
       const index = nextIndex++;
       results[index] = await fn(items[index], index);
     }
   }
   ```

4. **Spawn `min(limit, items.length)` workers and wait for all of them.**
   `Array.from({ length: workerCount }, () => worker())` starts the pool; with 8 items and
   `limit: 3` you get 3 workers → the concurrency check sees `maxActive` hit exactly 3 and
   never 4. `await Promise.all(workers)` — every worker resolves normally (they *catch*
   nothing; they simply stop when the queue is empty), so `Promise.all` only rejects if a
   `fn` call rejects, which is exactly the fail-fast behavior you want.

5. **Validate the limit at the door.** `limit: 0` would spawn zero workers and silently
   return an array of holes; `1.5` would silently floor to 1. A `RangeError` turns a silent
   wrong answer into a loud right one — and the self-check
   `rejects an invalid limit` pins that behavior down.

6. **Write the self-checks before you trust it.** The five checks that matter: order
   preservation (`deepEqual` against `items.map(n => n * 2)`), a concurrency meter (`active`/
   `maxActive` counters inside the test's own `fn` — assert `<= limit` *and* `=== limit`, so
   the limiter is proven both to cap and to actually run concurrently), `limit: 1` acts
   sequential, empty list returns `[]`, and a throwing item rejects the whole call.

The same shape — *naive version → name the bug → choose shared state → make failure loud →
prove it with an assertion* — is how the other three utilities were built.

## Concepts worth explaining

**1. The event loop, task queue vs microtask queue (the output-order puzzle).**

JavaScript runs one call stack. Async work doesn't run "in the background" — it finishes
elsewhere and its continuation gets *queued*. Two queues, drained in strict order: after
each synchronous run-to-completion, the event loop empties the **microtask queue**
(promise callbacks, `await` continuations) *completely*, and only then takes **one** task
from the **task queue** (`setTimeout`, I/O) and runs it — then drains microtasks again.

The checkpoint snippet produced this order, and why:

```text
A: sync start → B: inside go, before await → G: sync end
→ C: first .then → F: after await → E: second .then
→ D: setTimeout
```

- **A B G** — sync code runs to completion; `go()` pauses at `await null` and control
  returns to the script, which prints G.
- **C** first because its `.then` was queued before `go()` ever hit the `await`.
- **F before E** — the subtle one: `await` queues go()'s continuation *at the moment the
  await is evaluated* (during the sync phase), so F is already in the queue when C runs.
  C's handler queues E only at that point, i.e. *behind* F.
- **D last** — `setTimeout(..., 0)` is not "immediately", it is "in the next trip around
  the loop, after every microtask already queued has drained".

**2. `Promise.all` vs `allSettled` vs `race` vs `any`.** They share a shape — combine
promises, return one — but answer different questions. `all` = "did *everything* succeed?"
(first rejection wins, siblings are ignored-but-handled → the default for dependent work).
`allSettled` = "what happened to each?" (never rejects → batch jobs where partial failure
is data, not an error). `race` = "who finishes first?" (outcome *and* timing are the
point — this is what `withTimeout` uses, racing the real promise against a timeout). `any`
= "did *anyone* succeed" (first fulfillment wins, only rejects when every promise rejects
→ aggregating redundant fallbacks). Choosing: need all results → `all`; need a report →
`allSettled`; need first-event semantics → `race`; need first-success → `any`.

## What tripped me up

- **`sleep` threw synchronously while the other utilities rejected.** The first self-check
  run came back **14/15**, and the failure was my own validation: `sleep(-1)` threw its
  `RangeError` before `assert.rejects` ever got a promise to inspect (the error message in
  the failure output *was* the RangeError — the check had failed for the wrong reason). A
  utility that throws sync for bad input but rejects for bad timing is a trap for callers.
  Fix: made `sleep` an `async function`, so every validation failure in the toolkit arrives
  the same way — as a rejection. This is the forgotten-`await` family of bugs in a new
  costume: *inconsistent failure surfaces are bugs even when each path individually works.*
- **The first sequential timing run read 2236ms** — with a warm-up fetch already done, I
  assumed both measurements would look like later runs (~132ms/100ms). They didn't: one
  slow round-trip inside the loop adds its full latency to the total, because sequential
  waits for *every* response before moving on. Parallel doesn't get cheaper — it hides the
  outlier inside the overlap. It's now the clearest argument in the write-up for why
  parallel wins on networks and why unbounded parallelism still needs a limiter.
- **Deciding "is this file the entry point?" took real thought.** `runSelfChecks` must run
  under `node async-toolkit.js` but *not* when the timing script imports `sleep`. There is
  no single blessed idiom across Node versions, so the guard uses `import.meta.main` when
  available with a `fileURLToPath(import.meta.url)` vs `path.resolve(process.argv[1])`
  fallback — the fallback is what runs on this cohort's pinned Node if `main` isn't
  there. Worth knowing: getting this wrong doesn't fail loudly; it either re-runs 15 checks
  on every import or silently runs none.
- **Node version drift, caught by probing.** The training-ground pins Node 24 in `.nvmrc`,
  but a fresh shell here answered `v22.23.2` — the version manager (fnm on this box) had
  not auto-applied the pin. Ran every check with `fnm exec --using=24` to make the evidence
  match the pinned cohort version. (M01's pattern: verify by how the tool answers, not by
  what the config file says should be true.)

## Checkpoint evidence

**Self-checks — `node async-toolkit.js` on Node v24.21.0 (15/15, exit 0):**

```text
sleep
  ok - resolves after the requested delay
  ok - rejects a negative delay instead of resolving
retry
  ok - returns the value once fn succeeds
  ok - rethrows the last error when attempts run out
  ok - waits delayMs between attempts
  ok - rejects invalid options instead of hiding them
withTimeout
  ok - passes through a value that settles in time
  ok - rejects with TimeoutError when the promise is too slow
  ok - propagates the real rejection, not a timeout
mapLimit
  ok - maps every item, preserving input order
  ok - never runs more than `limit` calls at once
  ok - runs sequentially when limit is 1
  ok - handles an empty list
  ok - rejects when any call rejects
  ok - rejects an invalid limit

15/15 self-checks passed
```

**Timing comparison — `node timing-comparison.js`, 8 posts, three runs:**

| Run | Sequential | Parallel | Winner |
| --- | --- | --- | --- |
| 1 | 2236ms | 39ms | parallel by 2197ms (98%) |
| 2 | 132ms | 37ms | parallel by 95ms (72%) |
| 3 | 100ms | 36ms | parallel by 64ms (64%) |

```text
Fetching 8 posts from JSONPlaceholder

sequential      100ms  (8 posts)
parallel         36ms  (8 posts)

parallel beat sequential by 64ms (64% faster)
```

**Output order — predicted before running, actual run:**

Predicted: `A → B → G → C → F → E → D`. Actual:

```text
A: sync start
B: inside go, before await
G: sync end
C: first .then (microtask)
F: after await
E: second .then (microtask)
D: setTimeout (macrotask)
```

Seven for seven. The full prediction-with-reasoning and the run transcript are in the
training-ground repo's local `evidence/m02/` folder (gitignored — the PR history is the
public trail).

**fs refactor** — both versions print identical output:

```text
1. fix the flaky timing test
2. ship the toolkit
3. tease apart the event loop
4. write the M02 write-up
```

**Definition of done:** all `node:assert` self-checks pass ✔ · output order predicted and
explained out loud (above) ✔ · parallel beats sequential with an explanation (run 1's
2236ms outlier is the explanation) ✔ · branch → PR → merge habit continued:
[PR #3](https://github.com/juzzthynperez/node-js-training-ground/pull/3) ✔.

## What I'd do differently

- **Take the median of five timing runs, not the first three.** Run 1 (2236ms) is a great
  teaching anecdote but a poor central estimate; five runs with the middle three reported
  would separate "parallel wins" (certain) from "by exactly how much" (noisy).
- **Give `retry` a backoff option from the start** — `delayMs` fixed keeps the first
  version honest, but a `{ backoff = 'none' }` flag with the self-check extended to assert
  the growing delays would show the production shape without obscuring the basic one.
- **Make the concurrency meter reusable.** The `active`/`maxActive` counters were written
  inline in two separate self-checks; a tiny `makeConcurrencyMeter()` helper inside the
  checks would remove the duplication and model test-helper thinking for the next trainee.
- **Interleave the write-up more strictly.** I wrote each section right after its utility
  passed rather than in one end-of-day sitting, but the README's bar is "while you build" —
  next module I'll fill each section before moving to the next build step, so the first
  surprise lands in "What tripped me up" while it is still fresh, not reconstructed from
  the self-check output.
