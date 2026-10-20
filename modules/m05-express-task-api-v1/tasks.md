# M05 Tasks — Express: Routing, Middleware & Error Handling (Task API v1)

Build a well-structured REST API with Express 5 in TypeScript, with a consistent error model.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.
> The error envelope and the Task API's field/scope contract in `brief.md` are fixed program-wide —
> implement them, don't redesign them.

## Setup

- [x] Read this stage's `brief.md` fully, including the resolved Task API contract (error envelope
      + field/scope spec).
- [x] Confirm `@types/express` is installed and your project still type-checks under M04's
      `tsconfig.json`.
- [x] Continue the branch → PR → merge habit for this Friday-gate submission.

## Research

- [x] HTTP/REST refresher: methods, status codes (200/201/204/400/401/403/404/409/422/500),
      headers, JSON, idempotency, resource naming, `/api/v1` versioning, pagination conventions.
- [x] Why `app.ts` (builds/exports the app, no `listen`) is kept separate from `server.ts` (calls
      `listen`) — look up why this separation matters for testing later.
- [x] Typing Express handlers with `Request<Params, ResBody, ReqBody, Query>`.
- [x] Core routing: `app.get/post/put/patch/delete`, `express.Router()`, route params and query
      strings, separating handlers from route definitions, `res.status().json()`, `res.sendStatus`.
- [x] Read the official Express 5 migration guide, specifically: automatic forwarding of rejected
      promises, the new path syntax (`/*splat`, optional segments with `{}`), `req.body` being
      `undefined` without a body parser, `req.query` being a read-only getter, and removed
      deprecated signatures.
- [x] Middleware fundamentals: the pipeline, why order matters, `next()` vs `next(err)`,
      app-/router-/route-level middleware, and the built-ins (`express.json`, `express.urlencoded`,
      `express.static`).
- [x] Third-party middleware you'll likely want: `cors`, `helmet`, and `morgan` or `pino-http`.
- [x] Operational vs programmer errors, and the 4-argument error-handling middleware signature.
- [x] Config from environment variables, graceful shutdown on `SIGTERM` via `server.close`, and a
      health endpoint.
- [x] CORS basics, and `express-rate-limit` as a stretch goal if time allows.
- [x] Bruno: collections, environments, and assertions.

## Build

- [x] Write `app.ts` (exports the app, no `listen`) and `server.ts` (`listen` + graceful shutdown on
      `SIGTERM`) as two separate files from the start.
- [x] Build your `AppError` hierarchy (e.g. `BadRequestError`, `NotFoundError`, `ConflictError`) and
      a central 4-argument error handler that produces exactly the fixed envelope shape from the
      brief — no stack traces in responses.
- [x] Build the request-ID middleware (so `requestId` in the envelope is real, not a placeholder)
      and a logging middleware.
- [x] Build the in-memory task store.
- [x] Build `GET /api/v1/tasks` and `POST /api/v1/tasks`, with hand-written validation
      (deliberately — no Zod yet).
- [x] Build `GET`, `PATCH`, and `DELETE /api/v1/tasks/:id`, with hand-written validation.
- [x] Add a 404 handler for unmatched routes.
- [x] Add a health endpoint.
- [x] Wire CORS basics into the middleware stack.
- [x] Build the Bruno collection with assertions for every endpoint and every error path, and
      commit the `bruno/` folder.

## Verify

- [x] v1 is merged by pull request.
- [x] Every error path — not just one — returns the exact envelope shape from the brief.
- [x] The full Bruno collection passes.
- [x] You can explain, specifically, why Express 5 no longer needs a try/catch wrapper around async
      handlers.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": how you implemented the fixed error envelope and
      your `AppError` hierarchy, your middleware order, and why you hand-wrote validation here
      instead of reaching for a library.
- [x] "How to build it (teach it to the next trainee)": write the central-error-handler guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": Express 5 migration surprises, middleware ordering bugs, etc.
- [x] "Checkpoint evidence": v1 merged by PR, every error path matching the fixed envelope, and the
      Bruno collection passing.
- [x] Close out "What I'd do differently" — flag here if the fixed envelope or Task field/scope
      contract caused friction for your situation.

---

> Ticked 2026-10-20 by the trainer flow: ground solution merged (training-ground PR #10,
> `a51bd89`), `/validate` passed 20/20 (live typecheck, 15/15 Bruno, 17/17 mechanical
> envelope checks). `express-rate-limit` stretch not built — optional per the brief.
