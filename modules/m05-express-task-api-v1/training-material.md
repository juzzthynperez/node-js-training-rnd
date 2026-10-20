# M05 — Express: Routing, Middleware & Error Handling

> **Week 2 · Days 3–5 · 12 hours** · Builds on M03 (bare `node:http`) and M04 (strict
> TypeScript) · Task API **v1** — the foundation for M07, M08, and M09

## What you're building

You build **Task API v1** — a well-structured REST API in Express 5 and TypeScript with
one consistent error model. Everything in Modules 7, 8, and 9 grows out of what you build
here, so the goal is not "make five endpoints respond" — it's "make five endpoints
respond *predictably*, with errors a program can read."

By the end you will have a running API, a committed collection of API tests you can
re-run any time, and — the part that actually matters later — a single place where every
error becomes the same clean JSON.

## The domain, in one breath

**The task we build throughout:** a task tracker. A task has an id, a title, an optional
description, and a status that is one of `todo`, `in_progress`, or `done`. That's the
whole domain, on purpose — the interesting part of this module is the *structure*, not
the data.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string (UUID) | generated for you, used in every `/tasks/:id` URL |
| `title` | string | required, non-empty |
| `description` | string | optional |
| `status` | `todo` \| `in_progress` \| `done` | one of exactly these three |

## The five endpoints you'll end up with

| Method | Path | Does | Success |
| --- | --- | --- | --- |
| `GET` | `/api/v1/tasks` | list every task | `200` |
| `POST` | `/api/v1/tasks` | create one | `201` |
| `GET` | `/api/v1/tasks/:id` | read one | `200` |
| `PATCH` | `/api/v1/tasks/:id` | update part of one | `200` |
| `DELETE` | `/api/v1/tasks/:id` | remove one | `204` |

Behind them: a request-ID middleware, a logging middleware, and a single central error
handler that returns **every** error in one fixed JSON shape. Plus a committed Bruno
collection that verifies all of it.

## How the 12 hours break down

| # | Section | Focus |
| --- | --- | --- |
| 1 | Introduction to Express and REST APIs | the request–response lifecycle, HTTP, REST |
| 2 | Setting Up an Express Application with TypeScript | `app.ts` vs `server.ts`, typed handlers |
| 3 | Routing and Building REST Endpoints | paths, params, `Router()`, Express 5 changes |
| 4 | Middleware and the Request Pipeline | order, `next()`, custom request-ID + logging |
| 5 | Error Handling and the Consistent Error Envelope | `AppError`, the central handler, the fixed shape |
| 6 | Designing and Implementing the Task API v1 | the store, the five endpoints, hand-written validation |
| 7 | Application Configuration and Operational Basics | env config, CORS, health, graceful shutdown |
| 8 | Testing the API with Bruno | collections, environments, assertions |
| 9 | Final Integration and Verification | the gate run and the pull request |
| 10 | Wrap-up and Self-Assessment | explain it back, reflect |

## What's required, and what's just good to know

The brief's scope is bigger than what's assessed. Know which is which before you start:

| You must **build** it (assessed at the checkpoint) | You must **understand** it (discussed, not built) |
| --- | --- |
| `app.ts` (no `listen`) + `server.ts` (`listen` + `SIGTERM` shutdown) | HTTP methods, status-code families, idempotency, pagination conventions |
| `AppError` hierarchy + central 4-argument handler + the exact envelope | Express 5 path syntax (`/*splat`, `{}` optional segments) |
| Request-ID + logging middleware | `morgan` / `pino-http` (we build ours by hand) |
| In-memory store + all five `/api/v1/tasks` endpoints | `express.urlencoded`, `express.static` (not used here) |
| Hand-written validation (no libraries) | `helmet` — recommended, one line, not assessed |
| 404 handler + `GET /health` + CORS wired | `express-rate-limit` — **stretch only** |
| Bruno collection covering every endpoint and every error path | API-key auth middleware — optional practice, not assessed |
| `tsc --noEmit` clean, work merged by PR | `PUT` — explained below; v1 uses `PATCH` instead |

**How to read the code in this guide.** `ts` blocks are written to compile under this
project's strict config — except where a block is marked *illustrative* (shape only:
names like `app` or `store` stand for what you've built, and short snippets omit
`import type` lines — add `import type { Request, Response, NextFunction } from "express"`
where you use them). Nothing here is a drop-in solution: blocks show the shape and the
one idea being taught; the **Build it** boxes say what yours must do, and you write the
bodies.

---

## 1. Introduction to Express and REST APIs

### What Express does in a Node.js application

In Module 3 you built a server on the bare `node:http` module. It worked, and it taught
you what a web server actually does: listen on a port, get a request, send a response.
It also taught you how much boilerplate that takes. Recall the pieces you had to write
by hand: reading and parsing the request body, matching a URL to behavior, sending the
right status code, and — the one that hurts — wrapping every handler so one thrown error
didn't kill the whole process.

Express is a small layer over `node:http` that does that plumbing once, then hands you a
clean function to run. Where the bare module gave you `req` and `res` objects to
interpret yourself, Express gives you *middleware*: small functions that each handle one
concern and pass the request along.

Think of Express less as a framework and more as a **well-organized relay race**. The
request enters at one end, a line of runners each does their part, and the response
leaves at the other. Most of what you learn this module is about who runs, in what
order, and what happens when someone drops the baton.

### Understanding the request–response lifecycle

Every request follows the same path:

```
client sends request
        │
        ▼
  [request-ID middleware]   attach an id we can track
        │
        ▼
  [logging middleware]      note that a request happened
        │
        ▼
  [body parser]             turn the JSON body into a real object
        │
        ▼
  [your route handler]      decide status code + body
        │
        ▼
  [error handler]           (only if something threw)
        │
        ▼
client receives response
```

Two things to hold onto. First, the request passes through *every* middleware in order
before it reaches your handler — middleware can inspect the request, modify it, or end
the response right there. Second, at any point, any of these functions can fail, and
the failure travels to a different place: the error handler. Keeping those two paths
("the happy path" and "the error path") separate in your head will save you hours.

### HTTP methods, status codes, headers, and JSON

A request carries a **method** — the verb saying what you want done:

| Method | Meaning | Success status |
| --- | --- | --- |
| `GET` | Read a resource (or list) | `200` |
| `POST` | Create something new | `201` |
| `PATCH` | Change part of a resource | `200` |
| `PUT` | Replace a resource whole | `200` |
| `DELETE` | Remove a resource | `204` (no body) |

Status codes tell the client *how it went* without them reading the body:

- **2xx — it worked.** `200` OK, `201` Created, `204` No Content.
- **4xx — the client sent something wrong.** `400` Bad Request (your input is invalid),
  `404` Not Found (no such resource), `409` Conflict (your request clashes with what's
  already there).
- **5xx — the server broke.** `500` Internal Server Error. This one is never the
  client's fault, and never the client's business — see the error section for why.

An API **body** is JSON: JavaScript object notation, text on the wire, parsed into a
real object by middleware before your handler sees it. A request with no body (`GET`,
`DELETE`) simply has nothing to parse.

### REST resource naming and /api/v1 versioning

REST names things by *what they are*, not *what you want to do to them*. Compare:

```
# Not REST — verbs in the URL
POST /api/v1/createTask
POST /api/v1/deleteTask

# REST — nouns, HTTP method carries the verb
POST   /api/v1/tasks
DELETE /api/v1/tasks/:id
```

`/api/v1` is a **version** in the path. When you eventually make breaking changes, a
client can pin to `/api/v1` while a newer one uses `/api/v2`, and both keep working.
It costs nothing now and spares you later.

### Idempotency and basic pagination conventions

**Idempotency** means: doing the same request twice has the same effect as doing it
once. `GET` and `DELETE` are naturally idempotent — deleting an item twice leaves the
same result (it's gone). `PUT` is designed to be idempotent — setting a task's title to
the same value twice changes nothing the second time. `POST` is *not* idempotent —
creating twice makes two things. This matters more in Module 9 than today, but the
concept earns its place now: it's why retrying a `POST` is riskier than retrying a
`GET`.

**Pagination** is how a list endpoint avoids returning ten thousand rows at once. The
convention we follow: a list endpoint returns a fixed shape, and clients ask for a page
with query parameters. For v1 our list endpoint returns everything (our dataset is
tiny), but you'll see the `?limit=` / `?offset=` convention named in later modules —
knowing it exists is enough here.

### Understanding the Task API v1 deliverable

The endpoints and middleware you're building are in the overview at the top of this
document — that table is the target, and section 9 checks you against it. What the
overview doesn't tell you yet is *how* errors come back, because that part is fixed for
you rather than designed by you.

The error shape is **fixed program-wide** — you implement it, you don't redesign it:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [{ "path": "title", "message": "Required" }],
    "requestId": "3f2b…"
  }
}
```

`code` is a short machine-readable string. `message` is human-readable. `details` shows
up **only** on validation errors and is left off entirely otherwise. `requestId` ties
the response back to a log line. Modules 7, 8, and 9 all speak this shape — get it
right once and you're done with it.

---

## 2. Setting Up an Express Application with TypeScript

### Installing and configuring Express 5 with TypeScript

Start a fresh folder for the module. One install pass covers both halves — runtime and
types:

```bash
npm install express@5                       # the framework
npm install --save-dev @types/express typescript @types/node tsx
```

Express 5 is the version to use — it changed meaningfully from Express 4 (we'll meet
those changes in section 3), and most tutorials you'll find online describe Express 4,
so knowing you're on 5 helps you read them skeptically.

The step called **Installing @types/express** deserves its own note: Express is written
in JavaScript, so
TypeScript can't see its types natively. Without the types package,
`import express from 'express'` type-checks as `any` and you've quietly lost every
benefit of TypeScript. With it, `req`, `res`, and `next` all have real types.

### Creating app.ts and server.ts

This is one of the module's most important structural decisions: **two files, not one**.

`app.ts` builds the application — every middleware, every route — and *exports* it. It
never starts a server.

```ts
// app.ts
import express from "express";

export function buildApp() {
  const app = express();
  // middleware and routes go here — sections 3 through 6
  return app;
}
```

`server.ts` imports that app and *only* starts listening:

```ts
// server.ts
import { buildApp } from "./app.ts";

const app = buildApp();
const server = app.listen(3000, () => {
  console.log("listening on 3000");
});
```

A hard-coded `3000` is fine for right now — section 7 moves it to an environment
variable. The point to hold onto is *where* the number lives, not its value: only
`server.ts` ever opens a port.

### Understanding why application setup and server startup are separated

Because of one thing you will do in Module 9: test the API without starting a server.

When your tests import the *app* directly and hand it to a testing tool, every test can
say "here's a request, here's what I expect back" — no port opened, no real network, no
"start the server, wait, remember to stop it, hope the port wasn't busy" dance. If
`app.ts` called `listen` as a side effect of being imported, importing it would start a
server during your test run, and that falls apart fast.

Separating them costs two extra lines today and buys the entire Module 9 approach.

### Configuring express.json()

A request body arrives as raw text. To have `req.body` as a real JavaScript object, add
the built-in body parser as middleware:

```ts
app.use(express.json());
```

Express 5 detail worth knowing: if this middleware *isn't* in your stack, `req.body` is
`undefined` — not `{}`, not null, `undefined`. Reaching for `req.body.title` when no
parser ran throws, which is exactly the kind of confusing error the error handler in
section 5 exists to catch cleanly.

### Typing Express request handlers with TypeScript

Express's default types leave `req.params`, `req.body`, and `res.json(...)` loose. The
`Request` generic tightens them. The signature is
`Request<Params, ResBody, ReqBody, Query>` — four slots, all optional:

```ts
import type { Request, Response } from "express";

app.get("/api/v1/tasks/:id", (req: Request<{ id: string }>, res: Response) => {
  const { id } = req.params;          // typed as string — no cast needed
  res.json({ data: { id, title: "example" } });
});
```

The `{ id: string }` in the first slot is *you* telling Express what the URL segment is
named. If you later write `req.params.taskId`, the compiler flags it — that's the whole
payoff. You don't need every slot every time; we'll use all four when we build the real
routes in section 6.

### Running and verifying the application

`tsx` runs TypeScript directly — no build step, no watch mode to babysit. But an app
with no routes answers nothing, so give yourself one temporary route to prove the wiring
works:

```ts
// inside buildApp(), for now — section 3 replaces this with real routes
app.get("/", (_req, res) => {
  res.json({ status: "ok", note: "placeholder until section 3" });
});
```

Then run it:

```bash
npx tsx src/server.ts
```

In another terminal:

```bash
curl http://localhost:3000/
```

### Build it — requirements for this section

- A `src/app.ts` that exports a `buildApp()` function returning an Express app, with
  `express.json()` registered and your placeholder route. **No `listen` in this file** —
  that's the graded invariant.
- A `src/server.ts` that imports `buildApp`, calls `listen`, and is the only file in the
  project that does.
- A `tsconfig.json` with the strict flags carried from M04 (`strict`,
  `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, `erasableSyntaxOnly`,
  `allowImportingTsExtensions`, `noEmit`) plus `module`/`moduleResolution: nodenext`.
  Copy from the M04 module folder and adjust `include` for your new `src/` layout.
- Dependencies: `express@5` and `@types/express` at runtime-ish, `typescript`,
  `@types/node`, `tsx` as dev dependencies.

**If it doesn't work:**

- `tsc` reports `any` on every `req`/`res` → `@types/express` isn't installed or isn't
  picked up. Check `npm ls @types/express`.
- `tsx: command not found` or a missing esbuild binary on a fresh clone → npm now skips
  postinstall scripts by default; run `npm install-scripts ls`, then
  `npm install-scripts approve <package>` for what your tools need.
- `req.body` is `undefined` in a handler → `express.json()` isn't registered, or it's
  registered *after* the route. Order matters (section 4).
- Port already in use → an earlier run of `server.ts` is still alive; kill it.

> **Checkpoint:** Start the server and successfully return a JSON response. A `curl` in a
> second terminal prints a JSON body from your running app.

---

## 3. Routing and Building REST Endpoints

### Defining routes with app.get(), app.post(), app.patch(), and app.delete()

A route pairs an HTTP method and a path with a handler function:

```ts
app.get("/api/v1/tasks", (req, res) => { /* … */ });
app.post("/api/v1/tasks", (req, res) => { /* … */ });
app.patch("/api/v1/tasks/:id", (req, res) => { /* … */ });
app.delete("/api/v1/tasks/:id", (req, res) => { /* … */ });
```

Each handler receives the request, sends the response, and returns. The pair — method
plus path — is what identifies the route; `GET /api/v1/tasks` and
`POST /api/v1/tasks` are two different routes on the same path.

### Understanding route parameters and query strings

**Parameters** are parts of the path that vary, marked with a colon:

```ts
import type { Request, Response } from "express";

app.get("/api/v1/tasks/:id", (req: Request<{ id: string }>, res: Response) => {
  const { id } = req.params;   // string — you typed the slot in section 2
});
```

A request for `/api/v1/tasks/abc-123` matches the route with `id` set to `"abc-123"`.

**Query strings** are key-value pairs after a `?`, and they don't affect which route
matches:

```
GET /api/v1/tasks?status=done&limit=10
// req.query is { status: "done", limit: "10" }
```

Worth noting for Express 5: `req.query` is now a **read-only getter**. In Express 4
you could assign to it; in 5 you can't, and if you try, it fails loudly rather than
silently doing nothing.

### Returning appropriate status codes with res.status().json() and res.sendStatus()

Two tools for sending a response:

```ts
res.status(201).json({ data: task });      // a status code plus a JSON body
res.sendStatus(204);                        // just a status code, no body
```

`res.status(...)` returns `res`, so it chains. `res.sendStatus(n)` sends the code and
nothing else — the right choice for `DELETE`, where "it's gone" needs no body.

Match the code to the truth: `201` when you created something, `200` when you returned
or changed something, `204` when you deleted it. Your Bruno assertions in section 8 will
check exactly these, so getting them right here makes that section fast.

### Organizing routes with express.Router()

Once a file has more than a handful of routes it gets unreadable. `Router()` gives you a
group of routes you can mount at a path:

```ts
// routes/tasks.ts
import { Router } from "express";
export const tasksRouter = Router();

tasksRouter.get("/", (req, res) => { /* … */ });
tasksRouter.post("/", (req, res) => { /* … */ });
```

```ts
// app.ts
import { tasksRouter } from "./routes/tasks.ts";
app.use("/api/v1/tasks", tasksRouter);
```

The router's paths are relative to where it's mounted — `tasksRouter.get("/")` here
handles `GET /api/v1/tasks`. This is how a large API stays navigable: one router file
per resource.

### Separating route definitions from request handlers

A step further than organizing: put the *route table* and the *handler functions* in
different files. `routes/tasks.ts` declares **where** requests go; a
`task-handlers.ts` holds **what** each one does.

```ts
// routes/tasks.ts — the "where"
tasksRouter.get("/:id", handlers.get);
tasksRouter.patch("/:id", handlers.update);
```

```ts
// task-handlers.ts — the "what"  (illustrative — your strict config wants real types:
// export function get(req: Request<{ id: string }>, res: Response) { … } )
export function get(req, res) { /* … */ }
export function update(req, res) { /* … */ }
```

The payoff is readability at a glance: opening `routes/tasks.ts` shows you the API's
entire surface — every path and method — without a single handler's logic in the way.

### Understanding Express 5 routing changes and migration considerations

Express 5 changed routing syntax, and this matters because most online examples are
Express 4. The changes you'll actually hit:

- **Wildcards changed.** Express 4's `'*'` catch-all is now written `'/*splat'`. A bare
  `'*'` throws in Express 5.
- **Optional segments use `{}`.** `/users{/:id}` matches both `/users` and `/users/42`.
- **Removed signatures.** Some deprecated method overloads are gone —
  `res.json(status, body)` (passing a status as the first argument) was removed;
  `res.status(...).json(...)` is the way now.

When you read a tutorial and something won't match, check the Express version first —
it's the usual culprit.

### Build it — requirements for this section

- A router module (e.g. `src/routes/tasks.ts`) mounted at `/api/v1/tasks` in `app.ts`,
  with all five routes declared (handlers can be stubs for now — section 6 fills them).
- At least one working handler proving the wiring: reads a `:id` param with
  `Request<{ id: string }>`, responds with `res.status(...).json(...)`.
- Route definitions and handler functions live in separate files.

**If it doesn't work:**

- `GET` to a route you wrote returns your own 404 → check the mount path
  (`app.use("/api/v1/tasks", router)` + `router.get("/")` means `router.get("/:"…)`
  paths are *relative*; `/` on the router is `/api/v1/tasks` on the app).
- Copied `'*'` from a tutorial and the app won't start → Express 5 wants `'/*splat'`.
- `res.json(404, {…})` type-errors → that signature was removed in Express 5; write
  `res.status(404).json({…})`.
- Your route never fires because a catch-all or earlier middleware swallowed the
  request — registration order strikes again (section 4).

> **Checkpoint:** Build a basic task route and understand how requests reach the
> appropriate handler. A `GET` to your task path returns what you expect, and you can
> trace — out loud — how the request got there.

---

## 4. Middleware and the Request Pipeline

### What middleware is and how the request pipeline works

A middleware is a function with three arguments: request, response, and `next`
*(illustrative — the shape, before your types are attached)*:

```ts
function myMiddleware(req: Request, res: Response, next: NextFunction) {
  // do something with req or res
  next();            // pass control to the next middleware
}
```

`next()` is the baton. Call it and the request moves to the next function in line; don't
call it and the chain stops right there. Register them in order with `app.use(...)`
*(illustrative — `first`/`second`/`third` stand for your middleware functions)*:

```ts
app.use(first);
app.use(second);
app.use(third);
// request flows first → second → third → your handler
```

### Understanding middleware execution order

Order is not a detail — it **is** the program. Consider what depends on what:

- Request-ID must run **before** logging, because the log line prints the request ID.
- The body parser must run **before** your handlers, because they read `req.body`.
- The error handler must run **after** the routes, because errors thrown in a route
  need somewhere later in the line to land.

Get the order wrong and you don't get a compiler error — you get subtle bugs. A log
middleware registered before request-ID logs a request with no ID. It compiles. It runs.
It's just wrong. This is the single most common source of "it works but I don't know
why" in Express apps, and it's worth slowing down here.

### Using next() and next(err)

There are two ways to pass the baton, and the difference is the whole module in
miniature:

- `next()` — "all good, continue to the next **normal** middleware."
- `next(err)` — "something failed; skip ahead to the **error** handler."

`next(err)` with an argument is how the error path starts. It doesn't unwind like an
exception through a `catch` — it *jumps forward* to the first error-handling middleware
in the line, which is why the error handler must be registered after everything that
might fail.

### Application-level, router-level, and route-level middleware

The same middleware can be attached at three scopes *(illustrative — `mw` and `h` are
placeholders for real functions)*:

```ts
app.use(mw);              // application-level: runs for every request
router.use(mw);           // router-level: runs for requests hitting this router
route.get(path, mw, h);   // route-level: runs for this one route only
```

Pick the narrowest scope that does the job. A request-ID middleware is
application-level — every response should carry an ID. A "check this resource exists"
middleware is router-level. Something only one endpoint needs is route-level.

### Built-in middleware: express.json(), express.urlencoded(), and express.static()

Express ships with a few ready-made:

- `express.json()` — parses JSON request bodies. You've used this already.
- `express.urlencoded()` — parses form-submitted bodies. Not needed for a JSON API.
- `express.static()` — serves files from a folder. Not needed here either, but it's how
  you'd host a frontend from the same server.

### Third-party middleware: cors, helmet, and morgan or pino-http

The ecosystem fills the gaps:

- **`cors`** — required here; section 7 explains what it's for and wires it in.
- **`helmet`** — security headers, one line, recommended but not assessed.
- **`morgan` / `pino-http`** — request logging. We build our own (above, on purpose, so
  you see how it works); knowing these exist is what matters — a real team would reach
  for one.

### Building custom request-ID and logging middleware

Two small middleware that do real work for us — and one small piece of TypeScript
plumbing they both depend on.

**The plumbing.** `req.requestId` isn't part of Express's type definitions, so under the
strict config every use of it is a TS2339 error until you declare it once, in your
request-ID module:

```ts
// request-id.ts — declare global augments Express's Request for the whole project
declare global {
  namespace Express {
    interface Request {
      requestId: string;
    }
  }
}
```

Ambient declarations like this are erased at compile time, so they're compatible with
`erasableSyntaxOnly` — you're describing types, not emitting runtime code.

**Request ID** — every request gets an ID: use the client's if they sent one (that's
what makes correlation work), otherwise generate one; either way send it back as a
header:

```ts
import { randomUUID } from "node:crypto";
import type { NextFunction, Request, Response } from "express";

export function requestId(req: Request, res: Response, next: NextFunction): void {
  const inbound = req.get("x-request-id");          // client's id, if any
  req.requestId = inbound && inbound.trim() !== "" ? inbound.trim() : randomUUID();
  res.setHeader("x-request-id", req.requestId);
  next();
}
```

The echo behavior is deliberate and section 8 tests it: when a client (or a future you,
reading logs) sends `x-request-id: bruno-runner-42`, the response carries *that* value —
so the `requestId` inside an error envelope always matches what the client already knew
it sent.

**Logging** — one structured line per request when it finishes *(illustrative: same
shape, but keep the types from the request-ID module's imports)*:

```ts
export function logging(req: Request, res: Response, next: NextFunction): void {
  const start = Date.now();
  res.on("finish", () => {
    console.log(JSON.stringify({
      requestId: req.requestId,
      method: req.method,
      path: req.originalUrl,
      status: res.statusCode,
      durationMs: Date.now() - start,
    }));
  });
  next();
}
```

Note the `res.on("finish", ...)` — the log must be written *after* the response is sent,
because only then do we know the status code and timing. And note that it reads
`req.requestId` — which only works because request-ID was registered first. Order, again.

### Build it — requirements for this section

- A request-ID middleware that echoes a non-empty inbound `x-request-id` or generates a
  UUID, sets it as a response header, attaches it to `req.requestId`, and calls `next()`.
- The `declare global` block that gives `req.requestId` a type (one place, one time).
- A logging middleware emitting one JSON line per finished request including
  `requestId`, `method`, `path`, `status`, `durationMs`.
- Both registered in `app.ts` as the **first two** `app.use(...)` calls.
- Proof: send a request with `x-request-id: my-trace-1`, get `my-trace-1` back in the
  response header, and see it in the log line.

**If it doesn't work:**

- Log line shows `"requestId": undefined` → logging registered before request-ID, or
  request-ID calls `next(err)`/doesn't call `next()` at all.
- TS2339 `Property 'requestId' does not exist on type 'Request'` → the
  `declare global` block is missing, misspelled, or in a file that isn't a module
  (it must contain at least one import/export).
- Echo test fails with a random UUID → your middleware always generates; it must prefer
  the inbound header.

> **Checkpoint:** Requests receive a request ID and generate useful log output. Send a
> request, see the ID come back in a header, and see a matching log line.

---

## 5. Error Handling and the Consistent Error Envelope

### Understanding operational errors versus programmer errors

This distinction runs the whole section, so fix it early.

**Operational errors** are expected and part of normal running: a client sent a title
that's missing, asked for a task that doesn't exist, tried to create a duplicate. They
aren't bugs — they're the API doing its job, and their messages are safe to show a
client.

**Programmer errors** are bugs: dereferencing something that's `undefined`, a typo'd
variable name, a promise nobody awaited. Showing *these* to a client leaks your
internals — a stack trace names your files, your line numbers, your structure. And it
tells them nothing useful, because they can't fix your code.

Everything in this section flows from treating these two differently.

### Why centralized error handling matters

Without a central handler, every route writes its own error responses, and within a week
there are four different shapes in the codebase. Any client then has to parse errors
differently per endpoint — miserable.

With one, every error in the entire API becomes the same predictable JSON. The client
parses one shape, ever. That's the whole idea.

### Creating a custom AppError base class

Start by giving errors a class that carries everything the handler will need
*(class shape taught here; the constructor's job is only to stash the three fields)*:

```ts
// errors.ts — the shared detail shape and the base class live together
export interface ErrorDetail { path: string; message: string }

export class AppError extends Error {
  readonly status: number;
  readonly code: string;
  readonly details: ErrorDetail[];

  constructor(status: number, code: string, message: string, details: ErrorDetail[] = []) {
    super(message);
    this.name = new.target.name;
    this.status = status;
    this.code = code;
    this.details = details;
  }
}
```

Now the error itself knows its status code, its machine-readable `code`, and any
field-level `details` — the handler won't need a `switch` on error kinds, it just reads
fields. (Note `constructor(status: number, …)` — every parameter needs a type; with
`strict` on, the untyped version won't compile. Also note the field declarations *plus*
constructor assignments: parameter properties like `constructor(readonly status: number)`
are banned by `erasableSyntaxOnly`.)

### Extending it with errors such as BadRequestError, NotFoundError, and ConflictError

A subclass's whole job is pinning the status and code so a handler can throw the *kind*
of error. Here's one, complete:

```ts
export class BadRequestError extends AppError {
  constructor(message = "Request validation failed", details: ErrorDetail[] = []) {
    super(400, "VALIDATION_ERROR", message, details);
  }
}
```

A handler now reads as intent, not bookkeeping: `throw new BadRequestError("…", errors)`
says what happened; the class carries the 400 and the `VALIDATION_ERROR` code.

**Guided practice — write the other two yourself:**

| Class | Status | `code` | Default message |
| --- | --- | --- | --- |
| `NotFoundError` | `404` | `NOT_FOUND` | `"Resource not found"` |
| `ConflictError` | `409` | `CONFLICT` | `"Resource conflict"` |

They take only a message (no `details`) — a missing or conflicted resource is never a
validation error, and their envelopes must not grow a `details` field.

### Understanding Express's four-argument error middleware

Express tells error middleware apart from normal middleware **by its arity** — four
parameters, where the first is the error (typed here so it compiles; the *runtime*
recognition is purely the parameter count):

```ts
function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction) { /* … */ }
```

A middleware with three params is normal. With four, it's an error handler, and it only
runs when something earlier called `next(err)` or threw. This is easy to get subtly
wrong — if you declare three params, Express treats it as normal middleware and your
error handler silently never runs.

### Handling rejected promises in Express 5

Here is the Express 5 change that earns the version number. In Express 4, if an async
handler threw or its promise rejected, **Express didn't notice** — the rejection went
unhandled and the request hung. Every Express 4 codebase therefore wrapped handlers in
`try { … } catch (next) }` boilerplate.

Express 5 forwards rejected promises from async handlers to the error handler
automatically. So this just works — note the handler is `async` and throws *directly*,
no `await` needed to trigger the mechanism (the throw inside an async function *is* a
rejection):

```ts
// illustrative fragment — §6 adds the UUID guard and the real store wiring
// store.get() is synchronous today — the async keyword is the point, not the store
router.get("/:id", async (req: Request<{ id: string }>, res: Response) => {
  const task = store.get(req.params.id);
  if (!task) throw new NotFoundError(`Task ${req.params.id} not found`);
  res.json({ data: task });
});
```

No `try`, no `catch`, no wrapper. When M07 makes the store genuinely async and M08 adds
awaited validation, this same pattern keeps working unchanged — which is exactly why
it's taught now rather than in those modules.

### Returning the fixed JSON error envelope

This is where all of section 5 lands: **one** handler, **two** branches, one shape. The
rules it must follow — read these first, they're the requirements your code enforces:

1. **Guard first:** if the response already started sending (`res.headersSent`), you
   can't retroactively wrap it — call `next(err)` and let Express's default handler
   finish the job.
2. **`AppError` branch (operational):** take `status`, `code`, `message` from the error
   as-is — they're safe by construction. Include `details` **only when the array is
   non-empty**; on a 404 or 409 the field must be *absent* — not empty, not `null`.
   That's what the brief means by "empty/absent elsewhere," and clients rely on
   "`details` present" meaning "validation problem."
3. **Everything else (programmer error):** log the full error server-side — message and
   stack, where you can fix it — then send a fixed 500 whose message is a constant
   (`"Internal server error"`), **never** `err.message`. A bug's message like
   `Cannot read properties of undefined (reading 'title')` is a free map of your
   internals; that's the leak this whole branch exists to prevent.
4. **Both branches** attach `requestId` — it's what makes the envelope worth its weight:
   a client reports an error, you read their `requestId`, and you find the exact
   matching log line from section 4.

The one tricky mechanic — conditional inclusion of `details` — looks like this:

```ts
const envelope: ErrorEnvelope = { error: { code, message, requestId } };  // no details key
if (err instanceof AppError && err.details.length > 0) {
  envelope.error.details = err.details;   // added only for validation failures
}
```

Design note: express the envelope as an `ErrorEnvelope` interface (the `details` key
optional) and let *one* function build it for both branches. Two branches building JSON
by hand in two places is how shapes drift apart again — the thing you're trying to
prevent.

The production-safety rule (3) in one fragment:

```ts
if (!(err instanceof AppError)) {
  console.error(err);                    // full stack, server-side only
  message = "Internal server error";      // constant — err.message never reaches wire
}
```

### Avoiding stack traces and internal details in production responses

That was rule 3, stated once so nothing contradicts it — but it deserves its own name,
because it's the habit that outlives this module. The division of labor: **server logs**
get the full truth (message + stack, every error, including bugs); **the wire** gets an
`AppError`'s message as-is, and for anything else a constant `"Internal server error"`.
A stack trace in a response is a free map of your files and line numbers to anyone who
can trigger a bug, and it helps them not at all. If you want a development convenience,
you may echo the real message when `NODE_ENV !== "production"` — but that flag applies
only to *unexpected* errors; `AppError` messages are safe in every environment because
you wrote them.

### Handling unmatched routes with a 404 handler

A request for a URL no route matched never reaches a handler, so nothing sends a
response. Add a middleware *after* all routes that turns "nothing matched" into a normal
error:

```ts
app.use((req, res, next) => {
  next(new NotFoundError(`Route ${req.method} ${req.path} not found`));
});
```

It deliberately calls `next(err)` rather than responding — that way an unmatched route
produces *the same envelope* as any other 404. One shape, even for typos in the URL.

### Build it — requirements for this section

- `src/errors.ts` with the `AppError` base plus **your** `NotFoundError` and
  `ConflictError` (the table above), and `BadRequestError` — all three, typed.
- An `ErrorEnvelope` interface matching the fixed JSON from section 1 (`details`
  optional).
- A **4-argument** error middleware registered **after all routes**, implementing the
  four rules above — one shared envelope builder for both branches.
- A 404-for-unmatched-routes middleware registered between routes and the error handler,
  which throws/forwards a `NotFoundError` instead of responding.
- Proof of each branch: a validation failure (400 + `details`), a missing resource
  (404, no `details`), a conflict (409, no `details`), an unmatched route (404 envelope),
  and — for the programmer branch — a temporary `throw new Error("boom")` route that
  returns a clean 500 with a constant message while the full stack lands in your
  **server log**. Remove the temporary route when done.

**If it doesn't work:**

- Your error handler never runs → it has three parameters (Express needs four to
  recognize it), or it's registered *before* the routes it's supposed to catch.
- Error responses come back as Express's default HTML page → nothing forwarded to your
  handler; check position and arity.
- The 500 branch's real message appears on the wire → you're sending `err.message` for
  non-`AppError`s; rule 3 forbids it.
- `requestId` missing from envelopes → request-ID middleware isn't first, or it's not
  attaching to `req.requestId` (the `declare global` block from section 4).
- Envelope has `"details": []` on a 404 → you're including the key unconditionally;
  rule 2 says *absent*, not empty.

> **Checkpoint:** Trigger different errors and confirm that they each return the required
> error envelope. Try a missing field, a missing resource, and a URL that matches no
> route — every one comes back as the same JSON shape.

---

## 6. Designing and Implementing the Task API v1

### Reviewing the Task domain types from M04

You modeled these types already. Bring them forward:

```ts
type TaskStatus = "todo" | "in_progress" | "done";

interface Task {
  id: string;
  title: string;
  description?: string;        // optional
  status: TaskStatus;
}
```

The `status` union is doing quiet work: TypeScript will refuse to let you set a status
to anything outside those three values, at compile time, in every handler.

Derive the input types the M04 way — from `Task`, not by retyping the fields:

```ts
type CreateTaskInput = Omit<Task, "id">;
type UpdateTaskInput = Partial<Omit<Task, "id">>;
```

### Defining the API's Task fields and resource scope

For v1, the API exposes `tasks` and nothing else. A task has the four fields above —
`id`, `title`, an optional `description`, and `status`. Other data (users, tags) exists
in the database design from Module 6 but is deliberately *not* exposed as its own
endpoints here; it joins the API later. Keeping v1 to one resource keeps the learning
focused on structure.

### Creating an in-memory task store

No database yet — a `Map` holds the tasks, and a small class wraps it. This is a
*skeleton* *(illustrative — the three comment-only bodies won't compile until you write
them, which is the point)*:

```ts
import { randomUUID } from "node:crypto";

export class TaskStore {
  #tasks = new Map<string, Task>();

  list(): Task[] { return [...this.#tasks.values()]; }
  get(id: string): Task | undefined { return this.#tasks.get(id); }
  delete(id: string): boolean { return this.#tasks.delete(id); }

  create(input: CreateTaskInput): Task {
    // generate an id with randomUUID(), combine with input, store, return
  }

  update(id: string, patch: UpdateTaskInput): Task | undefined {
    // get existing (return undefined if missing), merge the patch, store, return
  }

  hasTitle(title: string, exceptId?: string): boolean {
    // true if any stored task has this title — ignoring exceptId when given
  }
}
```

Three things the shape is teaching: `#tasks` is a true private field (erasable — legal
under `erasableSyntaxOnly`, unlike parameter properties); `get` returning
`Task | undefined` is the honest signature — the task genuinely might not be there, and
that `undefined` is what makes `throw new NotFoundError(...)` necessary rather than
decorative in your handlers; and `hasTitle`'s `exceptId` exists because this section's
PATCH must allow a task to keep its *own* title.

One merge subtlety worth solving *before* you write `update`: a PATCH body that omits
`description` must preserve the stored one. A plain `{ ...existing, ...patch }` overwrites
with `undefined` when the patch carries `description: undefined` — decide how your merge
avoids that (checking `!== undefined` is the plain answer).

### Implementing the endpoints

Five endpoints on one router. Build them in this order — each one adds exactly one idea.
These are requirements, not code: your handlers are the deliverable.

- **`GET /api/v1/tasks`** — list. `200` with `{ data: … }` (an array). The simplest
  possible handler; use it to prove your router is wired.
- **`POST /api/v1/tasks`** — create. Validate, then `201` with the created task and a
  `Location` header pointing at it. Your first handler that can fail three different
  ways: bad input (400), a duplicate title (409), success (201).
- **`GET /api/v1/tasks/:id`** — read one. Check the id is a valid UUID (400 if not),
  then look it up (404 if missing). Two guards, in the order that matters: validate
  before you query.
- **`PATCH /api/v1/tasks/:id`** — update. Validate the body has at least one real field,
  apply the partial change, return the updated task. When the patch changes the *title*,
  check the new title doesn't collide with a **different** task's — the id you're
  editing is excluded from that check (that's what `hasTitle(title, exceptId)` is for).
- **`DELETE /api/v1/tasks/:id`** — remove. `204` on success; 404 if it was already gone.
  No body, because "it's gone" needs none.

Type requirement carried from section 2: every handler touching `:id` is typed —
`async (req: Request<{ id: string }>, res: Response)` — and none of them carry a
`try/catch`. If you're writing `catch` blocks, step back to section 5: a `throw` inside
an async handler already reaches the central error handler.

### Writing hand-written request validation

You are deliberately **not** using a validation library this module. Write the checks
yourself so you can see every piece — Module 8 replaces all of it with Zod, and you'll
appreciate the replacement because you'll remember what it replaced.

The contract: each validator takes an unknown body and returns either the cleaned input
or the field problems — declare that as its return type so callers can't misuse it
*(illustrative: the body below is deliberately incomplete — your final `return` of the
cleaned input or accumulated errors is the part you write)*:

```ts
// the shape your validators must return — ErrorDetail is { path, message }
type ValidationResult<T> = { input: T; errors?: never } | { errors: ErrorDetail[] };

export function validateCreateTask(body: unknown): ValidationResult<CreateTaskInput> {
  const errors: ErrorDetail[] = [];
  // your checks collect problems here — one example of the pattern:
  if (typeof body !== "object" || body === null || Array.isArray(body)) {
    return { errors: [{ path: "", message: "Body must be a JSON object" }] };
  }
  // … title required? status one of the three? description a string if present? …
  // … then: return { errors } if any, else return { input: cleaned } …
}
```

Write the rest: for `create`, a required non-empty `title`, a `status` that's one of the
three (defaulting to `"todo"` when absent), an optional string `description`. For
`update`, the field must be valid *if provided*, plus "at least one known field" — and
reject unknown field names outright.

The payoff to design for: the `{ path, message }` objects are exactly the envelope's
`details` format — so when a validator fails, the handler does
`new BadRequestError("Request validation failed", errors)` and the envelope comes out
already correct. Matching the wire format saves a translation step at exactly the wrong
moment (mid-error-response).

### Returning appropriate success and error status codes

Run through what each path owes the client:

| Situation | Status |
| --- | --- |
| List / get / update succeeded | `200` |
| Created | `201` |
| Deleted | `204` |
| Validation failed (bad field, bad id) | `400` |
| No such task | `404` |
| Duplicate title | `409` |
| Something unexpected broke | `500` |

### Build it — requirements for this section

- `src/task-types.ts` — `Task`, `TaskStatus`, `TASK_STATUSES`, plus the derived
  `CreateTaskInput`/`UpdateTaskInput`.
- `src/store.ts` — `TaskStore` with your three written method bodies; `get` keeps its
  `Task | undefined` return.
- Validators returning `ValidationResult<T>` — create and update, all hand-written.
- Five handlers, all `async`, all `:id` ones typed, **zero** `try/catch`, each producing
  exactly the status from the table.
- A duplicate-title rule on create, and on update *excluding the task being edited*.

**If it doesn't work:**

- `req.body` is `undefined` → `express.json()` missing or mis-ordered (section 2 trap,
  now real).
- Updating a task wipes its `description` → the merge-overwrite subtlety above; your
  merge must distinguish "absent" from "explicitly undefined."
- PATCH on a task reports a conflict with *itself* → your duplicate check didn't pass
  `exceptId`.
- A handler's `throw` returns HTML or hangs → that handler isn't async (or Express
  never forwarded it), or the central handler isn't registered after the router.
- `tsc` flags `body.title` on `unknown` → that's strict doing its job; narrow the type
  (`typeof` checks, as in the fragment) *before* reading properties — same `unknown`
  lesson as M04's catch blocks.

> **Checkpoint:** All five endpoints work, validate their inputs, and use the centralized
> error-handling model. Walk the table above with `curl` and see every row come out right.

---

## 7. Application Configuration and Operational Basics

### Reading configuration from environment variables

Anything that differs between your machine, a teammate's, and production belongs in the
environment, not in the code:

```ts
const port = Number(process.env.PORT ?? 3000);
```

Two habits worth forming now. Never hardcode a port — `PORT` lets whatever is running
you (your terminal, a cloud host, a test) decide. And read config in one file
(`config.ts`) that the rest of the app imports, so there's a single place that knows
where settings come from.

### Understanding CORS and configuring it for the API

CORS is the browser enforcing that a page running on one origin (say
`http://localhost:5173`) can't call an API on another (say `http://localhost:3000`)
unless the API opts in. It's a *browser* rule — `curl` and Bruno ignore it entirely,
which is why you might not notice its absence until a frontend won't load.

Adding it is one middleware: `npm install cors`, then `app.use(cors())` (and
`npm install helmet` if you're taking that recommendation; `express-rate-limit` if
you're attempting the stretch — neither is assessed). For a public API CORS is reflex.
For a same-origin API it's harmless. Knowing *why* it's there prevents the "works in
Bruno, fails in the browser" afternoon.

### Adding a health endpoint

Operations needs to know the API is up without touching business logic:

```ts
app.get("/health", (req, res) => {
  res.json({ status: "ok", uptime: process.uptime() });
});
```

Note it sits **outside** `/api/v1` — health is operational, not a versioned resource.
Load balancers and monitoring hit it constantly, so it should touch nothing expensive.

### Implementing graceful shutdown with SIGTERM and server.close()

When a server is told to stop, "stop immediately" can cut off requests that are
mid-flight. Graceful shutdown finishes them first:

```ts
server.close((err) => {
  process.exit(err ? 1 : 0);
});
```

`server.close()` stops accepting *new* connections and lets *in-flight* ones complete,
then calls back. Wire it to the signals a hosting platform sends:

```ts
process.on("SIGTERM", () => {
  server.close((err) => process.exit(err ? 1 : 0));
});
```

A note specific to this course: on some Windows setups, programmatically sending
yourself a signal doesn't run your handler — the process just ends. The handler above is
correct and is what you'd deploy on Linux; verifying the *signal path* end-to-end may
need a POSIX environment. Knowing the code is right and knowing the harness can't
demonstrate it are two separate facts, and it's worth being honest about which one
you've established.

### Understanding optional rate limiting with express-rate-limit

`express-rate-limit` caps how many requests one client can make in a window:

```ts
import rateLimit from "express-rate-limit";
app.use(rateLimit({ max: 100, windowMs: 60_000 }));
```

It's a **stretch goal** — one more `app.use(...)` if you have time. Skip it if you
don't; nothing later depends on it.

### Build it — requirements for this section

- `src/config.ts` owning environment reads (`PORT` at minimum, validated; a
  `NODE_ENV`-derived production flag if you follow the 500-message rule strictly);
  `server.ts` uses it instead of the literal `3000` from section 2.
- `app.use(cors())` in the stack, and helmet if you're taking the recommendation.
- `GET /health` returning `200` with a small JSON body (status + uptime is plenty),
  registered **before** the 404 middleware.
- `server.ts`'s `SIGTERM`/`SIGINT` → `server.close(...)` → exit wiring.

**If it doesn't work:**

- `/health` returns your 404 envelope → it was registered after the unmatched-route
  middleware, which ends the happy path for everything it sees. Health first, 404 last.
- `PORT` from the environment is a string and your listen or validation misbehaves →
  parse it (`Number(...)`) and check it's an integer in range; strings pass silently
  otherwise.
- CORS "works in Bruno but fails in the browser" → that's backwards from the usual
  confusion: Bruno/curl aren't browsers, so CORS never applied to your earlier tests.
  Test the browser case only when you have a browser page to test with.
- Your shutdown handler never fires on Windows → expected on this course's harness
  (the note above); on Linux it's the real path. Don't burn time chasing it here.

> **Checkpoint:** The API exposes a health endpoint and handles server shutdown cleanly.
> `curl /health` responds, and your shutdown handler is wired to `SIGTERM`.

---

## 8. Testing the API with Bruno

### Understanding Bruno collections and environments

Bruno is an API client — you build requests, run them against your server, and assert on
what comes back. Two ideas carry the section:

- A **collection** is a folder of saved requests, committed to your repo as plain text.
  Anyone who clones gets your API tests.
- An **environment** is a set of variables you swap. A `local` environment points at
  `http://localhost:3000`; a `staging` one points elsewhere. Your requests are written
  against `{{baseUrl}}`, and the environment decides what that is.

### Organizing requests by endpoint

Group requests the way the API is grouped — a `tasks` folder, an `ops` folder for
health. Name them so the story reads in the file tree: `01-list-tasks`,
`02-create-task`, `03-create-duplicate-409`. Ordered, descriptive names turn the
collection into documentation of your API's behavior.

### Testing successful responses

The happy path first — create a task, assert the response:

```
POST {{baseUrl}}/api/v1/tasks
{ "title": "Learn Express 5", "status": "todo" }

Expect: 201, body.data.title === "Learn Express 5"
```

When you create a task, save its id into a variable so later requests can use it —
`GET {{baseUrl}}/api/v1/tasks/{{createdTaskId}}`. **Where** you save it matters: the
script that reads the response must be declared `script:post-response { … }`, because a
plain `script { … }` block runs *before* the request, when there's no response to read.
Your collection then behaves like a real client: create, then use, then update, then
delete.

### Testing invalid input and missing resources

A happy-path-only suite proves almost nothing. The error paths are where your envelope
lives, so cover **every** distinct failure your API can produce — this list is the
required coverage:

- Create with **no title** → `400`, envelope carries `details`.
- Create with a **bogus status** → `400`, `details` names the allowed values.
- `GET` a task with a **malformed id** → `400`.
- `GET` a **well-formed but nonexistent** id → `404`, **no** `details` key.
- Create a **duplicate title** → `409`, no `details`.
- `PATCH` with an **empty object** and with an **unknown field** → `400` each.
- `DELETE` an **already-deleted** task → `404`.
- Hit a **URL no route matches** → `404` envelope, not a bare response.
- Success paths too: create returns `201` **and** a `Location` header; delete returns
  `204` with an empty body.

### Asserting status codes, response bodies, and error envelopes

Two kinds of assertion. The simple declarative form checks a status:

```
assert {
  res.status: eq 400
}
```

And a script test digs into the body to check the *envelope* — this is the assertion
that actually protects your contract:

```js
test("responds 400 with the fixed envelope", function () {
  const err = res.getBody().error;
  expect(err.code).to.equal("VALIDATION_ERROR");
  expect(err.details).to.be.an("array");
  expect(err.requestId).to.be.a("string");
});
```

Check the *shape*, not just the status: any handler can send a 400. Only a correct one
sends a 400 whose body matches your contract.

One real-world gotcha that will bite you: Bruno's script tests use **Chai** syntax —
`expect(x).to.equal(y)`, not Jest's `expect(x).toBe(y)`. If your test fails with
"Invalid Chai property: toBe," that's the fix. It's an easy mistake because Jest-style
assertions dominate most examples online.

### Testing request IDs and other relevant response fields

The `requestId` deserves its own assertion — it's the link between a client's error and
your logs:

```js
test("server echoes a client-supplied request id", function () {
  const rid = res.getHeaders()["x-request-id"];
  expect(rid).to.equal("bruno-runner-42");      // the value we sent in
});
```

Send an `x-request-id` header with your request and assert it came back unchanged. That
round-trip is what makes the envelope's `requestId` genuinely useful rather than
decorative.

### Committing the bruno/ collection to the repository

Bruno stores requests as plain text files. Commit the whole `bruno/` folder. It becomes
part of your deliverable — a reviewer clones the repo, starts the server, runs the
collection, and sees every endpoint and error path verify itself.

You can also run it headlessly, without opening the Bruno app. Install the CLI as a
dev dependency first — otherwise `npx bru` will go looking for *any* package named
`bru` and fetch an unrelated one:

```bash
npm install --save-dev @usebruno/cli
npx bru run --env local      # run from inside the bruno/ folder
```

### Build it — requirements for this section

- `@usebruno/cli` as a dev dependency, so headless runs work for anyone who clones.
- A committed `bruno/` folder: `bruno.json`, a `local` environment with `baseUrl`, and
  requests covering the full required-coverage list above (start the `tasks/` vs `ops/`
  split from section 9's layout).
- At least one envelope assertion that checks **body shape**, not just status —
  `code`, presence/absence of `details`, and a `requestId`.
- The request-ID echo test (client sends a known value, response carries it back).
- A green full run, headlessly or in the app, captured — section 9 wants it as evidence.

**If it doesn't work:**

- `You can run only at the root of a collection` → `cd bruno` first; `bru` must execute
  in the folder that holds `bruno.json`, not receive it as an argument.
- `npx bru` fails with usage text about *monorepos* (or anything that isn't Bruno) →
  `@usebruno/cli` isn't installed, so `npx` fetched the unrelated package named `bru`.
  Install the dev dependency and rerun from the collection root.
- `Invalid Chai property: toBe` → Bruno's `tests {}` use Chai (`expect(x).to.equal(y)`),
  not Jest. Rewrite the assertion chain.
- A test that reads the response runs against nothing → its script block is
  `script {}` (pre-request); move it to `script:post-response {}`.
- Everything 404s or connection-refused → server not running, or the environment isn't
  selected (`--env local`), so `{{baseUrl}}` resolved to nothing.
- Echo test returns a UUID you never sent → your middleware regenerates instead of
  echoing (section 4's rule).

> **Checkpoint:** The collection verifies every endpoint and relevant error path. The
> full run passes — every request green, including the error cases.

---

## 9. Final Integration and Verification

### Reviewing the complete application structure

Before you call it done, lay out the files and say what each is for:

```
src/
├─ app.ts                  # builds the app; every middleware, in order; no listen
├─ server.ts               # the ONLY file that calls listen; graceful shutdown
├─ config.ts               # reads PORT and friends from the environment
├─ errors.ts               # AppError + BadRequest/NotFound/Conflict + the envelope type
├─ store.ts                # in-memory TaskStore
├─ task-types.ts           # Task, TaskStatus — carried from M04
├─ routes/
│  └─ tasks.ts             # WHERE requests go (the route table)
├─ task-handlers.ts        # WHAT each route does
└─ middleware/
   ├─ request-id.ts        # attach/echo an id — runs FIRST
   ├─ logging.ts           # one JSON line per request
   ├─ validation.ts        # hand-written checks, errors as { path, message }[]
   ├─ not-found.ts         # unmatched routes → next(err) → same envelope
   └─ error-handler.ts     # central handler — runs LAST
```

If you can't explain why a file is where it is, that's worth ten minutes now.

### Running TypeScript type-checking

```bash
npx tsc --noEmit
```

Zero errors on the strict config, before you trust anything to run. This is the check
that catches the typo'd field name before a client does.

### Starting the server and running the Bruno collection

The gate run, in two terminals (with `@usebruno/cli` installed as a dev dependency,
per section 8):

```bash
npx tsx src/server.ts              # terminal 1
cd bruno && npx bru run --env local   # terminal 2
```

Every request green. If one fails, read which — the numbered names tell you where.

### Verifying the fixed error envelope across error responses

Re-check the contract holds *everywhere*, not just where you happened to test. Each
distinct failure shape — validation, not-found, conflict, unmatched-route — must come
back as exactly the envelope from section 1: `error.code`, `error.message`,
`error.requestId` always; `error.details` **only** on validation errors. Walk your error
paths once more and read the shapes, not just the status codes — this is the module's
headline guarantee; verify it deliberately.

### Reviewing middleware order and request flow

Read `app.ts` top to bottom and confirm the order still holds its invariants:
request-ID before logging (log has an id), body parser before routes (`req.body`
exists), error handler last (everything lands in it), 404 handler between routes and
error handler (unmatched routes get the envelope).

### Preparing the pull request and checking the module's completion requirements

Branch, commit, push, open the PR. Before you request review, confirm each:

- [ ] `app.ts` exports the app and never calls `listen`; `server.ts` is the only caller.
- [ ] Every error path returns the exact envelope.
- [ ] The Bruno collection passes in full.
- [ ] `tsc --noEmit` is clean.
- [ ] You can explain, out loud, why Express 5 needs no try/catch wrapper.

---

## 10. Wrap-up and Self-Assessment

Answer these in your own words — the explanations are the real deliverable.

**How does a request travel through the Express middleware and routing pipeline?**
Trace one real request from your own API. Name every middleware it passes, what each
one does, and where it would go if it failed instead.

**Why are app.ts and server.ts separate?**
Say what would break in Module 9 if they were one file. The answer is one sentence and
one idea.

**How does the centralized error handler produce consistent responses?**
From a `throw` in a handler to the JSON a client receives. Include where `details`
comes from and why it's absent on a 404.

**Why does Express 5 automatically forward rejected promises from async handlers?**
What broke in Express 4, what boilerplate it removed, and what you'd have to write
today if you were still on 4.

**Reflect on challenges, debugging lessons, and decisions made during implementation.**
What took longer than it should have? What surprised you? Where did you have to
re-read something to understand why it worked? That last one is usually where the
learning actually happened.

### What carries forward

Task API v1 is the foundation for the rest of the program. Module 7 replaces the
in-memory store with a real database. Module 8 replaces your hand-written validation
with Zod — the pain you felt writing it by hand is what makes that module land. Module 9
adds the automated tests that this module's structure makes possible.

The error envelope you built here travels through all of them, unchanged. Get it right
once and you're done with it.

---

*Part of the [node-js-training-rnd](https://github.com/juzzthynperez/node-js-training-rnd) training program · Practical work: [node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground)*
