# M05 Write-Up — Express: Routing, Middleware & Error Handling (Task API v1)

> Filled from the actual build in `node-js-training-ground` (PR #10, merge `a51bd89`).
> Environment: Node 24.21.0 through `fnm exec --using=24 -- …`, Express 5.3.0, TypeScript
> 7.0.2, `@types/express` 5.0.6, helmet 8.1.0, cors 2.8.6, `@usebruno/cli` for the
> headless collection run. This write-up is also my M07–M09 reference — written that way
> on purpose.

## What I built

Task API v1 under `modules/m05-express-task-api-v1/` in the ground repo:

- `src/app.ts` — builds and exports the app via `buildApp(config)`, never calls `listen`.
  The middleware stack, in order: `requestId` → `logging` → `helmet()` → `cors()` →
  `express.json()` → `GET /health` → `/api/v1/tasks` router → 404 handler → central error
  handler.
- `src/server.ts` — the only file that calls `listen`. Reads `loadConfig()`, logs
  `"listening"` as one JSON line, and wires `process.on("SIGTERM"/"SIGINT")` →
  `server.close(cb)` → `process.exit`.
- `src/errors.ts` — the `AppError` base (carries `status`, `code`, `details[]`) and three
  subclasses: `BadRequestError` (400/`VALIDATION_ERROR`, takes field details),
  `NotFoundError` (404/`NOT_FOUND`), `ConflictError` (409/`CONFLICT`). Plus the
  `ErrorEnvelope` interface — the literal wire shape from the brief, typed.
- `src/middleware/request-id.ts` — accepts an inbound `x-request-id` or generates a UUID,
  stashes it on `req.requestId`, echoes it back as a response header.
- `src/middleware/logging.ts` — one structured JSON line per request on `res "finish"`:
  method, path, status, durationMs, requestId.
- `src/middleware/error-handler.ts` — the central 4-argument handler. `AppError` → its
  status/code/message, `details` included **only when non-empty**. Anything else → 500
  with `INTERNAL_ERROR`; full error + stack logged server-side, and the response message
  is the real error only outside production.
- `src/middleware/not-found.ts` — unmatched routes call `next(new NotFoundError(...))`,
  so even the 404-for-no-route goes through the same envelope machinery.
- `src/middleware/validation.ts` — hand-written `validateCreateTask` / `validateUpdateTask`
  returning `{ input | patch, errors: ErrorDetail[] }`, plus a UUID check for `:id`.
- `src/store.ts` — `TaskStore` class over a `Map<string, Task>`; `hasTitle(title, exceptId?)`
  backs the 409 rule; `update` spreads the patch over the existing task.
- `src/task-types.ts` — the M04 domain types trimmed to the v1 contract: `Task { id,
  title, description?, status }`, `TaskStatus` union, `TASK_STATUSES` readonly array.
- `bruno/` — 15-request collection in two folders (`tasks/`, `ops/`) with a `local`
  environment; covers list/create/duplicate/validation-errors/get/patch/delete plus the
  unmatched-route 404, health, and the request-ID echo.
- `scripts/graceful-shutdown-test.ts` — the SIGTERM delivery repro (see "Checkpoint
  evidence" for why it exits 1 by design).

Gates: `tsc --noEmit` clean on the strict config; the full Bruno collection green
headlessly — **15/15 requests, 19/19 tests, 13/13 assertions**; all 16 curl scenarios in
`evidence/m05/error-path-captures.txt` show the exact envelope. PR #10 merged.

## Why it's built this way (key decisions)

- **The envelope is a type, then a handler, in that order.** `ErrorEnvelope` in
  `errors.ts` is the wire contract from the brief expressed as TypeScript; the central
  handler is the only code allowed to produce it. Handlers never touch `res` for errors —
  they `throw new BadRequestError(msg, details)` (sync) or let an async handler's promise
  reject (Express 5 forwards it). Two paths in, one shape out. The `...(details.length >
  0 ? { details } : {})` spread is what makes `details` *absent* (not `null`, not `[]`) on
  non-validation errors — the brief's "empty/absent elsewhere" honored as absent.
- **`AppError` subclasses, not a factory of `{status, code}` objects.** `instanceof
  AppError` in the handler is the cleanest runtime discriminator, and `new.target.name`
  gives honest `err.name` for logs. The alternative — a `makeError(status, code, msg)`
  factory — returns something the handler can't distinguish from a random Error without
  an extra brand field, which is a class with extra steps. M09's tests will also want
  `expect(err).toBeInstanceOf(NotFoundError)`-style assertions; classes give that free.
- **Middleware order is load-bearing, and I wrote it down as comments in `app.ts`.**
  `requestId` first — the logging middleware and the error envelope both *read*
  `req.requestId`, so moving it after `logging` means the first request's log line has no
  id, and moving it after the router means 400s from body parsing can't carry one.
  `errorHandlerFactory` last — Express only hands an error to error middleware that comes
  *after* the point of failure; put it before the routers and every thrown error falls to
  the default handler (HTML error page, no envelope). `notFound` sits between router and
  error handler so "no route matched" arrives as just another error.
- **409 lives in the store, validation stays shape-only.** `hasTitle` is a data rule
  ("no two tasks with the same title") and belongs with the data; `validation.ts` checks
  only types and presence. That seam is what M07 keeps: the store gets replaced wholesale
  by Prisma, and the duplicate check becomes a `P2002` mapping — validation shouldn't
  know anything about either.
- **Handlers are `async` even though the store is synchronous.** Two reasons: it makes
  the Express 5 "rejected promise → error handler, no try/catch" mechanism *real* in
  every handler rather than a claim in a comment, and when M07 swaps in Prisma (genuinely
  async) not one signature changes. The one wrinkle: `res.status(...).json(...)` inside
  an async handler can't meaningfully fail, but if it did (e.g. writing to a closed
  socket) Express 5's `headersSent` check in my handler catches it and defers to the
  default handler instead of double-sending.
- **Hand-written validation, on purpose.** `validateCreateTask` is ~20 lines of
  `typeof` checks returning the *same* `ErrorDetail[]` shape M08's Zod `issues` will
  produce. That's the payoff design: the error envelope never changes when the
  validation engine does — only `validation.ts` shrinks into `schemas/` + a `validate`
  middleware. Doing it by hand first is what makes that replacement legible instead of
  magical.
- **Request-ID accepts inbound.** A client (or Bruno, in `16-request-id-echo.bru`) that
  sends `x-request-id` gets *that* id back — the property that makes the envelope's
  `requestId` useful for correlating a trainee's Bruno run with server logs. Generating
  unconditionally would silently break that.

## How to build it (teach it to the next trainee)

**A central error handler that produces one consistent shape — the reasoning, not just
the code.**

Start from the failure it replaces: without a central handler, every route that wants a
404 writes its own `res.status(404).json({ message: '...' })`, and within a week there
are four different shapes in the codebase — `{ message }`, `{ error: '...' }`, `{ code,
msg }`, and one route that forgot and sent Express's HTML. Any client then needs
per-route parsing. The fix is a funnel:

1. **Decide the wire shape once** (here: the brief's envelope) and write it as an
   interface. The type is the contract; the handler is the only producer.
2. **Give your errors a class with the metadata attached.** `AppError` carries `status`,
   `code`, and `details` — the handler doesn't need a `switch` on error *kinds*, it just
   reads fields. Subclasses set sensible defaults, so a route throws
   `new NotFoundError(\`Task ${id} not found\`)` and nothing else.
3. **Register the handler last**, after `app.use(...)` of every router. Express
   recognizes error middleware by its *four-argument* signature — three args is a normal
   middleware, four is an error handler; getting this wrong silently changes behavior.
4. **Split operational from programmer errors.** `err instanceof AppError` → it's
   operational: the message is safe for the client, the status is meaningful. Not an
   `AppError` → programmer error: log everything (message + stack) *server-side*, respond
   with a fixed 500 that leaks nothing. The `NODE_ENV === "production"` check only tunes
   whether the *unexpected* message is echoed in development — an `AppError` message is
   always safe by construction.
5. **Honor `headersSent`.** If the response already started streaming (a body half-sent
   before an error struck), you cannot retroactively wrap it in an envelope — call
   `next(err)` and let Express's default handler close the connection.

Then wire the two entry points: sync handlers `throw`, async handlers just `reject` —
Express 5 forwards both. That's the whole trick: one funnel, one shape, and no route
ever touches the envelope by hand.

## Concepts worth explaining

**The middleware pipeline as a relay of `next()`.** Each middleware is a function with
`(req, res, next)`; calling `next()` passes the baton to the next one in registration
order, and that order *is* the program. It's not a bag of plugins — `requestId` before
`logging` because logging reads the id; body parser before routes because handlers read
`req.body`; error handler after everything because errors thrown anywhere before it must
reach it. Two kinds of baton-pass exist: `next()` ("all good, continue") and
`next(err)` ("something broke — skip every *normal* middleware ahead of me and go
straight to the first error handler"). A 404 handler in this model is just middleware
that always calls `next(err)` — which is why unmatched routes get the same envelope as
everything else.

**Operational vs programmer errors — why the split matters more than the codes.**
Operational errors are *expected*: a client sent a bad title, asked for a task that
doesn't exist, tried a duplicate. They're business as usual, their messages are safe to
show, and their statuses are part of the API contract. Programmer errors are *bugs*:
`undefined` dereferenced, a typo'd variable, a Promise someone forgot to await. Showing
those to a client leaks internals (stack traces name your file paths and line numbers);
logging them to the client wastes their time — they can't fix your code. The central
handler's `instanceof AppError` branch is exactly this boundary: one branch trusts the
error, the other branch *distrusts* it — full truth to the logs, a lie of omission
("Internal server error") on the wire. This is also why "make every error an `AppError`"
would be the wrong fix: it would launder bugs into 400s.

## What tripped me up

- **Bruno's `tests {}` blocks are Chai, not Jest.** My first headless run: all 13
  declarative `assert` blocks passed, 18 of 19 script tests failed with
  `Invalid Chai property: toBe. Did you mean "to"?`. Bruno's docs examples happen to use
  Chai-style chains and I'd written Jest-style (`expect(x).toBe(y)`); the fix was
  mechanical (`to.equal`, `.to.be.a("string")`, `.to.be.undefined`, `.to.include`) but
  cost a run. Related gotchas in the same session: `bru run` must be executed *at the
  collection root* (the folder with `bruno.json`) — passing the folder as an argument
  errors with "You can run only at the root of a collection"; a script that reads the
  response must be `script:post-response {}` (plain `script {}` runs *before* the
  request, so `res.getBody()` is undefined there); and environment files accept `vars`
  but reject an `auth` block outright.
- **The SIGTERM evidence problem, again.** The M03 retro predicted exactly this decision
  ("decide early whether a POSIX run or a code-review-scoped claim is acceptable"). A
  fourth delivery path — spawning the server as a child and calling
  `child.kill("SIGTERM")` — produced the same result as M03's three: libuv terminates
  the process unconditionally (`code=null`, no `shutdown` log, handler skipped). WSL's
  distro has no Node and Docker Desktop's engine wasn't running, so no POSIX run either.
  Scoped claim accepted: the handler is wired and reviewed, the signal *path* is not
  verifiable here; repro kept in `scripts/graceful-shutdown-test.ts`, exit 1 is the
  expected outcome on this harness.
- **`npm install` silently skips postinstall scripts now.** esbuild (tsx's engine) came
  down with its postinstall unapproved and tsx failed to resolve its binary until I ran
  `npm install-scripts approve esbuild`. Two prompts, two approvals (esbuild,
  protobufjs-for-bruno), and the installs completed. Any trainee hitting "tsx: command
  not found" on a fresh clone should check this first.
- **Express 5's async forwarding removed a whole category of bug *silently*.** Writing
  handlers, I kept reaching for `try { ... } catch (next) }` — muscle memory from every
  Express 4 tutorial — and had to consciously stop. The one behavioral trap worth
  teaching: it forwards *rejections*, so a truly synchronous throw inside a non-async
  handler is still caught by Express (it wraps handler invocation), but mixing the two
  styles in one file makes the error path hard to reason about; I kept every handler
  async for uniformity.

## Checkpoint evidence

- **v1 merged by PR:** [training-ground#10](https://github.com/juzzthynperez/node-js-training-ground/pull/10),
  merge `a51bd89`, `--delete-branch` used, no stray m05 branches.
- **Every error path returns the exact envelope:** `evidence/m05/error-path-captures.txt`
  — 16 scenarios (6 success, 10 error-path); the five 400 shapes (missing title, bad
  status, bad `:id`, empty patch, unknown field) all carry `details[]`; the four 404
  scenarios (missing task on GET and on DELETE, double delete, unmatched route) and the
  409 all carry `code`/`message`/`requestId` with no `details`.
- **Bruno collection passes:** `evidence/m05/bruno-run.txt` — 15/15 requests, 19/19
  tests, 13/13 assertions, exit 0, run headlessly from the collection root against a
  live server on the `local` environment.
- **Type-check:** `evidence/m05/typecheck.txt` — `tsc --noEmit` clean on the strict
  config.
- **Scoped claim (honest):** `evidence/m05/graceful-shutdown.txt` — the SIGTERM handler
  is wired and code-reviewed; the signal path is not end-to-end verifiable on this
  Windows harness (4th manifestation of the M03 finding).

## What I'd do differently

- **The 409-on-duplicate-title rule itself deserves a second look.** It was the obvious
  way to make `ConflictError` real, but title-uniqueness as a hard rule will fight M07's
  seed script and any trainee who wants two "Buy milk" tasks. If starting over: make the
  conflict a *status-transition* rule (can't move a done task back to todo? — any
  domain rule works) or accept duplicate titles and drop 409 from the required set. The
  contract in the brief fixed the envelope, not the domain rule — I picked one that may
  be more opinion than the program wants. Flagging for M07 when the seed script lands.
- **Validation's error-detail *shape* should have been pinned in the brief too.**
  `[{ path, message }]` was my choice (matching the envelope example), but Zod's native
  issues use `path` + `message` + more — M08 will need a mapper either way. Cheap to
  specify now, expensive to retrofit across a cohort.
- **`buildApp(config)` taking config as a parameter** was right for M09-testability, but
  I only arrived at it by rewriting `app.ts` halfway through (it used to read
  `loadConfig()` directly). Next cohort: start from the parameterized version — it also
  makes the config-vs-app separation teachable in section 2 of the training material
  instead of an afterthought.
