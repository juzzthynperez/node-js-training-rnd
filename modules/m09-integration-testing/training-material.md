# M09 — Integration Testing with Jest + Supertest

**Week 4 · Days 1–2 · 8 hours**

## 1. Introduction to Integration Testing

**What software testing is and why it matters in backend development.** A test is an executable
question about your code: you arrange a situation, do something, and assert what must be true.
In backend development the cost of *not* testing is paid by users — a broken endpoint doesn't
show a red squiggle in anyone's editor, it returns wrong data (or wipes the right data) in
production. Tests convert "I believe this works" into "this has been shown to work, and will be
shown again on every change".

**The testing pyramid.** Unit tests sit at the bottom: fast, numerous, testing one function in
isolation. Integration tests sit in the middle: fewer, slower, proving that several real parts
work together. End-to-end tests sit at the top: fewest, slowest, driving the whole system
(browser, network, external services). The pyramid's shape is advice about *distribution* —
lean on the cheap layers for breadth, keep the expensive layers for the paths that matter —
not a rule that integration tests are second-class. For an API like ours, most of the risk
*lives* in the integration layer: routing, validation, SQL, error mapping.

**Function tests vs request-to-database flow.** Testing `isNonEmptyString("x")` says nothing
about whether `POST /api/v1/tasks` with `{"title": " "}` returns 400 with the frozen message —
that answer is produced by five layers (route → validate middleware → Zod schema → handler →
service → Prisma → SQLite), and each seam between them is a place the bug hides. An integration
test crosses every seam in one shot: it sends an actual HTTP request through the actual
middleware chain and asserts what comes out.

**Why this module uses a real SQLite database instead of mocking.** A mock database is the
test author's *belief* about the database — you code the fake to return what you think Prisma
would return. But the behaviors we must prove only exist in the real engine: `P2002` is SQLite's
unique constraint *actually refusing* a duplicate title (the source of the 409); a FK violation
really happens only when the FK exists; "the DELETE persisted" can only be attested by a second,
fresh query. Mock those, and the suite proves only that your mock agrees with you. The brief's
rule is therefore absolute: **tests hit the real exported app and a real SQLite database — no
mocking the database.** The price of that realism is isolation machinery (section 4) — and
that price is exactly what makes the tests trustworthy.

**Reviewing Task API v3 and what the tests must prove.** v3 (M08) gave you: Zod validation
before every handler, the frozen error envelope (`{ error: { code, message, details?, requestId } }`),
Prisma error mapping (P2002→409, P2025→404, P2003→400), the v1 wire quirks (unknown keys
stripped on create but rejected on PATCH, duplicate-title checked *before* existence on PATCH,
the string-first page gate). The tests must *pin* all of that — every byte-exact message is a
contract line. If a test fails after a refactor, either the code broke or the contract drifted;
there is no third option, and finding out which is the test suite doing its job.

**Task API v4 and the role of automated testing in maintaining behavior.** v4 is v3 plus a test
suite plus one new endpoint (`PATCH /tasks/:id/complete`, written test-first). "Maintaining
existing behavior" is the key phrase: every M05–M08 behavior that Bruno asserted manually now
becomes an automatic assertion that re-runs in CI on every pull request. The suite is a *floor*:
future changes can add behavior, but they cannot silently subtract yours.

> **Checkpoint:** Explain what an integration test verifies and why the existing API is the
> starting point for this module.

## 2. Getting Started with Jest 30

**Installing the pieces.** Jest 30 is the runner; `@swc/jest` is the *transform* that converts
TypeScript to JavaScript on the fly (SWC is a Rust compiler — it strips types instead of
type-checking them); `@types/jest`/`@jest/globals` provide types; `supertest` (with
`@types/supertest`) drives HTTP. Install as dev dependencies:

```bash
npm install --save-dev jest@30 supertest@7 @types/supertest @swc/jest @swc/core
```

**The project's TypeScript/ESM configuration.** This project is `"type": "module"` with
TypeScript 7 and explicit `.ts` import specifiers (`import { buildApp } from "../src/app.ts"`) —
Node's ESM rules demand the extension, and `allowImportingTsExtensions` in `tsconfig.json`
permits it. Two consequences shape Jest's config:

```js
// jest.config.mjs — the config is .mjs so Jest can LOAD it without a transform
// (the file that configures the transform pipeline must not depend on it)
export default {
  testEnvironment: "node",
  extensionsToTreatAsEsm: [".ts"],
  transform: {
    "^.+\\.tsx?$": ["@swc/jest", { jsc: { parser: { syntax: "typescript" } } }],
  },
  testMatch: ["<rootDir>/tests/**/*.test.ts"],
};
```

And because Jest's ESM mode is still flagged, the run script carries the Node switch —
cross-platform (no Unix-only `NODE_OPTIONS=` prefix):

```json
"test": "node --experimental-vm-modules node_modules/jest/bin/jest.js --runInBand"
```

**Crucially: the transform is NOT type-checking.** SWC strips types; a test file with a type
error still compiles and runs. Types are enforced separately by `npm run typecheck`
(`tsc --noEmit`) in CI. Two jobs, two tools, both gates.

**`describe` / `it` / `expect` come from `@jest/globals`.** In ESM mode, import them explicitly
— there are no globals:

```ts
import { describe, it, expect, beforeEach } from "@jest/globals";
```

(`describe` groups, `it`/`test` names one behavior, `expect` carries the assertion. Note Jest's
mock utility object is called `jest` — Vitest's `vi` will not import here.)

**Suites, matchers, async.** Common matchers: `toBe` (primitives, identity), `toEqual` (deep
equality), `toMatchObject` (subset match — assert the fields you care about, not every one),
`toContain`, `toHaveLength`, `toBeUndefined`, plus `rejects`/`toThrow` for failures. Tests are
async by nature — `await` the request, then assert:

```ts
it("creates a task: 201 with the service-decided defaults", async () => {
  const res = await request(app).post("/api/v1/tasks").send({ title: "Write tests" });
  expect(res.status).toBe(201);
  expect(res.body.data).toMatchObject({ status: "todo", priority: "medium", dueDate: null });
});
```

**Arrange–Act–Assert.** Structure every test in three readable chunks: *arrange* (create the
rows, build the app), *act* (one request), *assert* (the contract). When a test is hard to
name, it is usually doing two acts — split it.

**`beforeAll` / `beforeEach` / `afterAll` — setup and cleanup.** `beforeAll` runs once per
suite (expensive shared setup), `beforeEach` runs before *every* test (state reset — the default
you want with a real database), `afterAll` runs once at the end (connections close:
`afterAll(() => prisma.$disconnect())`). This module's pattern:

```ts
beforeEach(async () => {
  await resetDatabase(); // FK-safe truncation — see section 4
});
```

> **Checkpoint:** Create and run an initial Jest test, with the test runner working in the
> project's ESM environment.

## 3. Testing Express Routes with Supertest

**Why the exported app needs no port.** `app.ts` exports `buildApp(config)` and never calls
`listen()` (that is `server.ts`'s one job) — a seam M05 deliberately drew for this moment.
Supertest takes the app object itself and drives requests through the full middleware chain
**in-process**: no socket, no port conflicts, no `localhost` lifecycle, fast enough to run
hundreds of tests per second.

```ts
import request from "supertest";
import { makeTestApp } from "./helpers/app.ts";

const app = makeTestApp(); // real chain: request-id → logging → helmet/cors/json → routes → 404 → error handler
const res = await request(app).get("/api/v1/tasks");
```

**The verbs and the request builders.** `.get/.post/.patch/.delete(path)` select the method;
`.send(body)` sets a JSON body (object auto-serialized); `.set(header, value)` sets headers
(`Content-Type` for malformed-JSON tests, `x-request-id` for middleware tests); `.query()`
appends query params — or inline them in the path (`"/api/v1/tasks?page=2&pageSize=2"`).

**Asserting status, headers, and body.** Every response exposes `res.status`,
`res.headers[...]` (note: header lookups are lower-cased), and `res.body` (parsed JSON):

```ts
const res = await request(app)
  .post("/api/v1/tasks")
  .send({ title: "spaced  " });

expect(res.status).toBe(201);
expect(res.headers["location"]).toBe(`/api/v1/tasks/${res.body.data.id}`);
expect(res.body.data.title).toBe("spaced"); // the schema trimmed it
```

**Async/await style** is the default: each `await` is one request. Express 5 forwards rejected
async handler promises to the central error handler automatically, so error responses are just
another awaited request — no try/catch games in tests either.

**Focused assertions: `toMatchObject` and `expect.objectContaining`.** Assert the fields the
test *claims* about, without re-listing the entire response — brittle over-assertion is how
suites rot:

```ts
expect(res.body).toMatchObject({
  error: { code: "NOT_FOUND", message: `Task ${id} not found` },
});
expect(res.body.error.details).toBeUndefined(); // absence IS part of this contract
```

**`test.each` — one behavior, many inputs.** The page gate rejects leading zeros, scientific
notation, zero, and non-numerics *for the same reason* — one table, one assertion:

```ts
it.each([["page=007"], ["page=1e3"], ["page=0"], ["page=abc"]])(
  "?%s → 400 'Must be a positive integer'",
  async (query) => {
    const res = await request(app).get(`/api/v1/tasks?${query}`);
    expect(res.status).toBe(400);
    expect(res.body.error.details).toEqual([
      { path: "page", message: "Must be a positive integer" },
    ]);
  },
);
```

(Agents/cookies exist for session flows — `.jar()` and `.agent()` share cookies across requests;
our API is cookie-less, so they stay unused here.)

> **Checkpoint:** Write an integration test that sends a request through an existing Task API
> route and verifies its response.

## 4. Setting Up an Isolated SQLite Test Database

**Why tests must never touch the development database.** Three failure modes: tests *pollute*
dev data (a test task shows up in your Bruno runs), dev data *corrupts* tests (yesterday's rows
make today's count assertions wrong), and worst of all, the suite's own cleanup **deletes** —
point it at `dev.db` and `resetDatabase()`'s `deleteMany` empties your real tables. Isolation is
not hygiene; it is data safety.

**`.env.test` and a separate `test.db`.**

```bash
# .env.test — the TEST environment (gitignored: committed env files are how secrets leak)
NODE_ENV=test
PORT=3000
DATABASE_URL="file:./test.db"
```

Add `.env.test` and `test.db` to `.gitignore`. Then confront the consequence: if the file is
not committed, a fresh CI checkout doesn't have it — so **the worker setup must force the
variable, not merely dotenv-load it** (below).

**Load the test environment BEFORE the Prisma client initializes.** `src/db.ts` reads
`DATABASE_URL` **at module load**, and its `import "dotenv/config"` would load `.env`
(`dev.db`) if nothing had set it. `dotenv` never overrides an already-set variable — which gives
you an ordering lever: Jest's `setupFiles` run **before the test file's imports are evaluated**,
so setting the variable there makes the later dev-env load a no-op:

```js
// tests/setup-env.mjs — runs per worker, before any test module is imported
import { config } from "dotenv";
config({ path: ".env.test", quiet: true });

// FORCE — no fallback, no escape hatch. .env.test is gitignored, so CI has no file
// to load; without this line db.ts falls back to file:./dev.db and the suite's
// deleteMany would empty development data.
process.env.DATABASE_URL = "file:./test.db";
process.env.NODE_ENV = "test";
```

**`prisma migrate deploy` prepares the schema — from recorded history.** A Jest `globalSetup`
runs once in the main process before any worker spawns:

```js
// tests/global-setup.mjs (plain .mjs — infra should not depend on the transform)
import { execFileSync } from "node:child_process";
import { rmSync } from "node:fs";

export default async function globalSetup() {
  // 1. rebuild the disposable test database from RECORDED migrations
  rmSync("test.db", { force: true });           // test.db is build output — safe to drop
  execFileSync(process.execPath, ["node_modules/prisma/build/index.js", "migrate", "deploy"], {
    env: { ...process.env, DATABASE_URL: "file:./test.db" },
    stdio: "inherit",
  });
}
```

Never `prisma migrate reset` here: it is destructive-by-default (consent-gated in agent
contexts per M08's pattern) and rewrites rather than replays. `migrate deploy` on a fresh file
is "rebuild from recorded history" — the same verb the seed path uses.

**Register it, plus the per-worker pieces:**

```js
// jest.config.mjs (excerpt)
globalSetup: "<rootDir>/tests/global-setup.mjs",
setupFiles: ["<rootDir>/tests/setup-env.mjs"],           // env first
setupFilesAfterEnv: ["<rootDir>/tests/setup-framework.ts"], // afterAll(prisma.$disconnect())
```

**Clean tables before each test — in FK-safe order.** Children first, always, because the
database *refuses* the wrong order:

```ts
export async function resetDatabase(): Promise<void> {
  await prisma.taskTag.deleteMany(); // junction references tasks AND tags
  await prisma.task.deleteMany();
  await prisma.tag.deleteMany();
  await prisma.user.deleteMany();
  // tasks.user_id is NOT NULL — re-provision the pinned API owner or every POST 400s (P2003)
  await prisma.user.create({ data: { id: DEFAULT_OWNER_ID, /* … */ } });
}
```

**Factories and seed helpers.** Reusable row builders with **deterministic** values — explicit
ids and timestamps wherever an assertion depends on ordering:

```ts
export function sequencedDate(i: number): Date {
  return new Date(Date.UTC(2026, 0, 1 + i, 12, 0, 0)); // index → one fixed instant
}
```

**`--runInBand` — one writer.** SQLite allows a single writer; multiple Jest workers on one
file fight over locks. The test script's `--runInBand` runs suites sequentially in one worker
with one `test.db`. (The alternative — a DB file per worker via `JEST_WORKER_ID` — trades
simpler cleanup for N schemas to keep in sync; we take the simple side.)

**Disconnect when done.** One `setupFilesAfterEnv` file registers it for every suite:

```ts
afterAll(async () => { await prisma.$disconnect(); });
```

**Verify isolation — don't assume it.** Two proofs worth keeping: a test asserting
`process.env.DATABASE_URL === "file:./test.db"` inside the worker, and one full run performed
with `.env.test` physically removed (the CI checkout simulation).

> **Checkpoint:** Run a test against the dedicated SQLite database, verify that test records are
> isolated, and confirm that the development database remains untouched.

## 5. Testing Successful API Behavior

**Cover every verb's success path first.** If the happy path is wrong, the error paths don't
matter. One test per endpoint, each pinning status *and* shape *and* persistence:

```ts
it("creates a task: 201, Location header, service-decided defaults", async () => {
  const res = await request(app).post("/api/v1/tasks").send({ title: "Write tests" });

  expect(res.status).toBe(201);
  expect(res.headers["location"]).toBe(`/api/v1/tasks/${res.body.data.id}`);
  expect(res.body.data).toMatchObject({ status: "todo", priority: "medium", dueDate: null });
  expect(await countTasks()).toBe(1); // persisted — not merely echoed
});
```

**Expected status codes and response shapes.** The contract: `201 + Location` on create,
`200 { data }` on reads/updates, `204` (empty) on delete, `200 { data, meta }` on the list.
Response *shape* assertions belong where they're claimed — `toMatchObject` for the fields under
test, `toEqual` for meta blocks whose exact values are frozen:

```ts
expect(res.body.meta).toEqual({ page: 1, pageSize: 20, total: 3, totalPages: 1 });
```

**Confirm persistence — query beyond the response.** A 201 only proves the handler *said*
"created". The database is the witness: count rows, re-fetch through a **separate** request,
assert the effect survived (see `countTasks()` above, and the delete test asserting `0`).

**Pagination, filtering, sorting.** Each is a table of arrange→act→assert with deterministic
fixtures (factories with sequenced timestamps):

```ts
await seedTasks(5);
const res = await request(app).get("/api/v1/tasks?page=2&pageSize=2");
expect(res.body.data.map(t => t.title)).toEqual(["task-002", "task-001"]); // createdAt desc
const filtered = await request(app).get("/api/v1/tasks?status=done");
expect(filtered.body.meta.total).toBe(1);
```

Also pin the *boundary* semantics: `pageSize` max 100, `totalPages = max(1, ceil(total/pageSize))`,
and the id-tiebreak that makes pages stable when timestamps collide.

**Boundary cases: empty results and absent params.** Empty must be `{ data: [], meta: { … total: 0 } }`
— a 200, never an error. Absent optional params are the *default* path: `?status=` (empty
string) must behave as *not supplied* (the M07 parity rule), and a queryless `GET /tasks` is the
most-run request in the suite:

```ts
const res = await request(app).get("/api/v1/tasks?status=&q=&page=&pageSize=");
expect(res.status).toBe(200); // absent semantics — empty means absent, not invalid
```

**Factories without hiding the behavior.** A good factory sets *inputs*; the test still asserts
*observable outputs*. If the assertion lives inside the helper, the test no longer shows the
reader what behavior it protects. Keep helpers dumb (`createTaskRow({...})` → a row) and keep
every `expect` in the test body where the story is.

**Assert behavior, not implementation.** No assertions on Prisma call counts, SQL strings, or
private methods — those break on refactors that change nothing users see. If a test would fail
when `task-service.ts` is rewritten with identical wire behavior, it's asserting the wrong
thing.

> **Checkpoint:** Cover the successful behavior of every existing endpoint and verify that the
> results match the API contract.

## 6. Testing Validation, Errors, and Middleware

**Invalid bodies, params, and query strings.** Every 400 assertion pins two things: the status
*and* the byte-exact frozen message in `details` — these strings are wire contract (Bruno and
the API's clients see them):

```ts
const res = await request(app).post("/api/v1/tasks").send({ description: "x" });
expect(res.status).toBe(400);
expect(res.body.error).toMatchObject({ code: "VALIDATION_ERROR", message: "Request validation failed" });
expect(res.body.error.details).toEqual([{ path: "title", message: "Required, non-empty string" }]);
```

**Zod failures → 400 with `details`.** The envelope rule: `details` is a *per-path* array, one
entry per actionable problem, and it is **absent** when empty. Test the classes distinctly —
unknown keys (`{ path: "foo", message: "Unknown field" }`, sorted first), type errors, enum
messages (`"Must be one of: todo, in_progress, done"`), and the mutual exclusivity rule
(`{}` → only the cross-field message; `{foo:1}` → only "Unknown field", never both).

**404 for resources and routes.** Two different 404s, both frozen:

```ts
// resource
expect(res.body.error).toMatchObject({ code: "NOT_FOUND", message: `Task ${id} not found` });
// unmatched route (funneled through the same central envelope)
expect(res.body.error.message).toBe("Route GET /api/v1/nonexistent not found");
// and the gate order: non-UUID id → 400 "Must be a UUID"; hex-shaped-but-not-RFC → 404
```

**409 on unique violations.** The DB's `@@unique([userId, title])` is the real engine saying
no — the test's arrange creates the duplicate, the act POSTs it, the assert pins the code and
message: `{ code: "CONFLICT", message: 'A task with title "…" already exists' }`. Also pin the
v1 quirk: PATCHing a *missing* id with a *taken* title answers **409** (clash checked before
existence), while a free title answers 404.

**Malformed JSON → controlled error.** Send a body the parser cannot read (no mock, no helper
— just a broken string with the right `Content-Type`):

```ts
const res = await request(app)
  .post("/api/v1/tasks")
  .set("Content-Type", "application/json")
  .send('{"title": ');
expect(res.status).toBe(400); // NOT 500 — a broken body is a client error
expect(res.body.error).toMatchObject({ code: "VALIDATION_ERROR", message: "Body must be valid JSON" });
expect(JSON.stringify(res.body)).not.toMatch(/stack|SyntaxError/);
```

**Request-ID middleware.** Three claims: it *sets* the header when absent (UUID format), it
*echoes* an inbound `x-request-id` unchanged, and the id reaches **error** responses too — both
as the header and as `error.requestId` in the body (that correlation is the whole point of the
middleware existing first in the chain):

```ts
const res = await request(app).get("/api/v1/tasks/…").set("x-request-id", "trace-err-456");
expect(res.headers["x-request-id"]).toBe("trace-err-456");
expect(res.body.error.requestId).toBe("trace-err-456");
```

**The error handler never leaks stack traces.** Force a programmer error at the sanctioned
boundary — spy on the service, never the database:

```ts
jest.spyOn(TaskService.prototype, "getById").mockRejectedValueOnce(
  new Error("unexpected failure in the service layer"), // message is echoed in dev BY DESIGN
);
const res = await request(app).get(`/api/v1/tasks/${row.id}`);
expect(res.status).toBe(500);
const raw = JSON.stringify(res.body);
expect(res.body.error.stack).toBeUndefined();
expect(raw).not.toContain(".ts:"); // a leaked stack names its frames' files
expect(raw).not.toContain("src/");
```

(The subtlety: non-production *echoes the message* deliberately — so the fixture's message must
itself be path-free, and the assertions must check what only a **stack** could carry.)

**`rejects` / `toThrow` for async failures.** For code paths that aren't HTTP-shaped (a helper
that should throw):

```ts
await expect(prisma.task.create({ data: danglingRow })).rejects.toThrow();
```

**Exact vs partial matching — when.** Exact (`toEqual`) for frozen envelopes and meta blocks:
those bytes are contract. Partial (`toMatchObject`) for objects that legitimately gain fields
later (a `data` payload): over-assertion makes the suite punish *additions*, and teams route
around punishing suites. Absence claims (`toBeUndefined()`) are exact by nature — write them
when the contract says a field must not appear.

> **Checkpoint:** Verify that the API handles expected failures consistently and that internal
> error details are not leaked to clients.

## 7. Test-Driven Development: Completing a Task

**The cycle.** *Red:* write a test that expresses the desired behavior and watch it fail.
*Green:* write the minimum that makes it pass. *Refactor:* improve the code **under the green
safety net**. The order is the method — a test written after the code merely describes what
already happened; a test written before the code is a *specification*.

**Define the endpoint's behavior first, on paper.** For `PATCH /tasks/:id/complete`:

- existing task → `200`, `{ data }` with `status: "done"`;
- only `status` changes — title, description, priority, dueDate untouched;
- completing an already-done task → **200 again** (idempotent, not an error);
- unknown-but-valid id → `404` through the same Prisma error seam (`Task <id> not found`);
- non-UUID id → `400` at the params gate; no body exists, so no body schema is needed.

**Write the integration tests before the endpoint.** It fails *because the endpoint doesn't
exist* — the intended reason:

```
RUNS  tests/patch.complete.test.ts
✕  marks an existing task done and returns the updated task
  Received: "Route PATCH /api/v1/tasks/00000000-…-000000000011/complete not found"
Tests: 5 failed, 5 total        ← evidence/m09/tdd-red.txt, captured BEFORE any implementation
```

**Confirm the failure is the *right* failure.** Read the message. `Route … not found` proves
the route is absent — good red. A `TypeError` in your assertion or a `400` from a mistyped path
is a *wrong* red: fix the test first. A red you can't explain is a red you can't trust when it
turns green.

**Implement only what the test demands.** Handler (params gate → service → `200 { data }`),
route (`router.patch("/:id/complete", validate({ params: idParamsSchema }), handlers.complete)`),
service method (one `update` setting `status: "done"`, errors through the existing
`mapPrismaError` seam):

```ts
async complete(id: string): Promise<Task> {
  try {
    const row = await this.prisma.task.update({ where: { id }, data: { status: "done" } });
    return toWire(row);
  } catch (err) {
    throw mapPrismaError(err, { id }); // P2025 → 404, same seam as every write
  }
}
```

Run: **green** (`evidence/m09/tdd-green.txt`).

**Refactor with behavior preserved.** The refactor step here had a real finding: the endpoint
needed **no new schema file at all** — `idParamsSchema` is the entire request contract — and the
route registration couldn't collide with `PATCH /:id` (one path segment vs two). Nothing about
the tests changed; the code got smaller. That's the cycle working: contract fixed first,
implementation shaped to it.

**The failure-path tests join the same first red.** Idempotent re-complete, fields-not-owned-
untouched, 404 with the exact message, 400 at the gate: all four were already failing in the
transcript above — they are part of the paper spec, not a batch written after green. Whole
spec red together, whole spec green together; written post-green they would have been retrofits
of the implementation instead of specifications of it.

> **Checkpoint:** Demonstrate that the completion endpoint was implemented test-first, with a
> failing test before the implementation and a passing test afterward.

## 8. Reliable Tests, Coverage, and Mocking

**Coverage with `--coverage`.** `npm run test:coverage` instruments the run and prints a
per-file table (statements / branches / functions / lines). Instrument *your* code only:

```js
collectCoverageFrom: ["src/routes/**/*.ts", "src/services/**/*.ts", "!src/generated/**"],
coverageThreshold: {
  "src/routes/**":    { statements: 80, branches: 80, functions: 80, lines: 80 },
  "src/services/**":  { statements: 80, branches: 80, functions: 80, lines: 80 },
},
```

**Thresholds are exit codes, not vibes.** With `coverageThreshold` in config, dropping below
80% **fails the run** (observed for real: initial branch coverage in `task-service.ts` was
74.35% and the run went red). `src/generated/**` stays excluded — generated code would pad the
percentage while proving nothing about yours.

**Reaching 80%+ on routes and services.** The usual gap is *unreachable arms*, not untested
endpoints. `mapPrismaError`'s P2003 case can't be reached through the tasks-only API (the owner
is a pinned constant) — the fix is neither lowering the threshold nor leaving a dead spot, but a
seam-level suite that provokes **genuine** Prisma errors from the **real** client (dangling-FK
insert, duplicate insert, update of a missing row) and asserts the mapping. Real database, real
errors, direct entry point — coverage as a forcing function for honesty.

**Flaky tests: shared state, time, ordering.** Three classic causes: *shared state* (test B
depends on rows test A left behind — fixed by full `beforeEach` reset), *time* (tests that pass
at 09:00 and fail at midnight — fix by injecting/stubbing time at the boundary, never by
sleeping), *ordering* (passing only in the sequence they were written). Flakiness is a bug in
the *setup*, not a nuisance to rerun until green — a red-flaky-red-green suite is teaching you
something every time it lies.

**`--randomize` exposes order dependence.** `npm run test:random` shuffles suite and test
order (each run with a seed). Any test that passes in default order but fails randomized was
reaching into another test's state — fix the missing reset or the hidden dependency, *never* the
flag. (This module: three randomized runs, 58/58 each — isolation built first, so there was
nothing to expose.)

**Mocking only at boundaries: `jest.fn`, `jest.spyOn`.** Mock what's *outside* the system under
test — or a collaborator you need to force into an unusual state. The suite's single spy:

```ts
jest.spyOn(TaskService.prototype, "getById").mockRejectedValueOnce(new Error("…"));
// …forces the 500 path; the DATABASE underneath stays completely real
```

`jest.fn()` wraps a function (count calls, stub returns); `jest.spyOn(obj, "method")` observes
or replaces one method while keeping the rest intact — and `jest.restoreAllMocks()` in
`afterEach` prevents leakage into the next test.

**Why the database is never mocked here.** Reprise of section 1 with the sharper edge: the
behaviors under contract — constraint violations, FK refusals, persistence, cleanup order —
*are* database behaviors. A mocked DB can only confirm your assumptions back to you. In this
codebase the database is *inside* the behavior under test; the boundary (service method forcing
an error) is where faking is legitimate.

**ESM mocking differs: `jest.unstable_mockModule`.** CommonJS `jest.mock(path, factory)` relies
on *hoisting* — it jumps above the imports that call `require()`. ESM has static imports and no
`require`, so hoisting is impossible: in ESM you call `await jest.unstable_mockModule(path, factory)`
**dynamically**, and the module must be imported *after* (usually via a dynamic
`await import(...)`) — the substitution takes only for consumers that import the module
*dynamically after* the mock call. That's why this module barely mocks: the
real-DB strategy minimizes the surface where ESM mock mechanics would even be needed.

**Snapshots — sparingly.** `expect(body).toMatchSnapshot()` stores a formatted copy and fails
on any diff. Useful for *large, stable, low-claim* payloads (a generated config dump); harmful
for API contracts — a snapshot assertion answers "did this change?" instead of "is this right?",
and the reflexive fix (`-u`) silently re-blesses regressions. This suite uses zero snapshots:
every response is asserted field by field, in words the contract can be read against.

> **Checkpoint:** Meet the required coverage target and resolve any test-order dependencies
> exposed by randomized execution.

## 9. Final Integration, CI, and Verification

**Review for endpoint and error-path coverage.** Walk the route table against the suite: every
route × (success, validation 400, 404, conflict 409 where applicable) — plus the cross-cutting
cases: malformed JSON, request-id on both answer paths, 404 handler, no-stack 500. If a route
or an error class has no test, it's not "covered by inspection"; it's uncovered.

**Confirm the two realness properties.** Every test builds the **real exported app**
(`buildApp` — no port, full chain) and runs against the **dedicated SQLite test.db**. Search
the suite for `jest.mock` over `db`/`prisma`/`task-service` returning canned data: there should
be none (the one `spyOn` forces an *error*; the DB beneath it is real).

**`.env.test` and `test.db` must not be committed.** Both are gitignored (plus the `test.db-journal` / `-wal` / `-shm` sidecars).
Verify mechanically, not by eye:

```bash
git check-ignore -v .env.test test.db   # each line must report an ignore rule
git ls-files | grep -E '\.env\.test|test\.db'   # must print nothing
```

**Run the full local sequence — exactly as CI will.** Four gates, in CI's order:

```bash
npx prisma generate     # fresh client (CI starts with none — it's gitignored)
npm run typecheck       # tsc --noEmit — includes tests/**
npm test                # ESM + runInBand against test.db
npm run test:random     # the same suite, shuffled
```

Plus the **CI checkout simulation**: rename `.env.test` away, run the suite, confirm still green
(this is the run that catches the forced-`DATABASE_URL` regression before CI does).

**GitHub Actions workflow.** One file at the *repo root* (Actions only reads
`.github/workflows/` there), scoped to the module directory:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: modules/m08-zod-validation   # the reused module dir
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm, cache-dependency-path: modules/m08-zod-validation/package-lock.json }
      - run: npm ci
      - run: npx prisma generate
      - run: npm run typecheck
      - run: npm test
```

`npm ci` (lockfile-exact install) → `prisma generate` (client is gitignored) → `typecheck` →
`test`. Node 24 matches `engines`.

**Green on the PR.** The workflow triggers on pull requests to `main`; the check appears on the
PR page and blocks merge while red. Push the branch, open the PR, watch all four steps — that
green run *is* the DoD's CI evidence (link it in the write-up).

**Status badge in the README.**

```markdown
[![CI](https://github.com/<owner>/<repo>/actions/workflows/ci.yml/badge.svg)](https://github.com/<owner>/<repo>/actions/workflows/ci.yml)
```

**Review the three claims, with evidence.** (1) test-first `PATCH /complete`: the red
transcript predates the implementation; (2) coverage report ≥ 80% on routes and services;
(3) randomized run green. Capture each output into `evidence/` with an index file mapping claim
→ transcript — an evidence claim nobody can locate is only an assertion.

**Submit through branch → PR → merge.** The established workflow: work on a feature branch,
green locally, push **with owner consent**, open the PR, pre-PR review, merge (owner). The
write-up's "Checkpoint evidence" section carries the PR link, the coverage table, and the
randomize output.

> **Checkpoint:** Confirm that the required tests pass locally and in CI, coverage is at least
> 80% for routes and services, and the suite passes with `--randomize`.

## 10. Wrap-Up and Self-Assessment

- **Unit vs integration vs end-to-end.** Unit: one unit, isolated, fastest, proves logic in the
  small. Integration: several real parts wired together (our app + middleware + Prisma + real
  SQLite), proves the seams — where this API's risk lives. End-to-end: the whole deployed system
  through its real interface, fewest and slowest. The pyramid advises distribution: breadth at
  the bottom, the critical paths proven at every layer.
- **Jest + Supertest together.** Jest is the runner — discovery, isolation, hooks, matchers,
  coverage, CI exit codes. Supertest is the request engine — it takes the *exported app* (no
  port) and drives real HTTP through the full middleware chain in-process, returning a response
  object the matchers assert against. One question per test: arrange, one awaited request, assert
  the contract.
- **Dedicated database and why cleanup matters.** The suite runs against `test.db` because a
  mock can't witness real constraints, and against a *reset* `test.db` because every test's rows
  are the next test's contamination — wrong counts, phantom 409s, positional order lies that
  `--randomize` turns into flakiness. FK-safe order (`task_tags → tasks → tags → users`) isn't
  pedantry: the schema *refuses* the wrong order, and the pinned owner row must be re-provisioned
  or every create 400s.
- **Successes and expected failures.** Happy-path tests pin status/shape/persistence; error
  tests pin status + code + the byte-exact frozen messages + absence of leaks (`details`
  omitted when empty, no stack, no `src/`). Malformed JSON is a client error (400), the 404
  funnel and resource-404 share the envelope, 409 is the DB's unique constraint speaking.
- **How test-first shaped the completion endpoint.** The red test forced the *contract* first —
  idempotence, which fields it doesn't own, which seam produces the 404, no body → no new schema
  — so the implementation had exactly one job: make five named expectations true. The code that
  emerged was smaller than the code that a spec-first approach would have guessed.
- **Flaky causes and randomization.** Shared state, time dependence, execution-order dependence —
  all fixable at the setup level (full reset, deterministic fixtures, boundary-only fakes).
  `--randomize` is the detector: order-dependence is a bug to fix, not a flag to avoid; a suite
  built with per-test isolation gives it nothing to find.
- **Transform vs type-check.** `@swc/jest` *executes* TypeScript by stripping types — fast, and
  blind to type errors. `tsc --noEmit` *checks* types without running anything. Both are gates:
  tests must run (Jest) and tests must type-check (`npm run typecheck` in CI). Fusing them (e.g.
  `ts-jest` type-checking every run) trades one job's clarity for the other's speed.
- **Coverage, isolation, CI satisfied.** Routes 100% / services 100% statements, 84.61%
  branches — over the enforced 80% threshold; 58 tests green in three randomized runs; suite
  proven green with `.env.test` absent (CI simulation); workflow runs the exact local sequence
  on every PR.
- **Reflection before M10.** What was hard: the ESM/infra ordering problems (env before module
  load — invisible until it silently points at the wrong database), the anti-leak assertion
  tripping over the intentional dev-mode message echo, and discovering unreachable coverage arms
  only at the gate. What was learned: assert *infrastructure invariants first*, distinguish
  "suite failed to run" (authoring bug) from "test failed" (behavior signal), and design
  coverage alongside the test plan. What to improve: capture evidence with an index as you go,
  and start the PR/consent conversation while the build is still being written.

> **Self-assessment:** Can you confidently write an integration test for a new API endpoint,
> isolate its database state, verify its error behavior, and run the test suite in CI?
