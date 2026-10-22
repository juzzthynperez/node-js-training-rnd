# M09 Write-Up — Integration Testing with Jest + Supertest (Task API v4)

> Working-tree draft written by the assistant; rnd git (commit/push) stays with the owner.

## What I built

**Task API v4** — the v3 codebase (M08) grown a real test suite, evolved **in place in
`modules/m08-zod-validation/`** (owner-directed revision: no copied module folder; v3's exact
state stays recoverable from branch `feat/m08-zod-task-api-v3` @ `887ecb8`), on branch
`feat/m09-integration-task-api-v4` (`5a84d7b` build + `bf31465` pre-PR review hygiene, PR #15):

- **58 integration tests / 5 suites**, all against the real exported `buildApp()` and a real
  SQLite `test.db` — happy paths for every endpoint, every frozen 400/404/409 envelope string,
  malformed JSON, the list-query int gate (`page=007`, `page=1e3` → 400), middleware behaviour
  (request-id on success **and** error responses, 404 funnel, no stack traces), and direct-client
  proofs of the `mapPrismaError` seam (P2002/P2003/P2025 → 409/400/404).
- **Test-database strategy**: `.env.test` + a Jest `globalSetup` that deletes the previous
  `test.db` and replays recorded migrations (`prisma migrate deploy` — never `reset`), a
  per-worker `setupFiles` that **forces** `DATABASE_URL=file:./test.db`, FK-safe
  `beforeEach` cleanup (`task_tags → tasks → tags → users` + owner re-provision), factories with
  deterministic ids/timestamps, `--runInBand`, `afterAll(prisma.$disconnect)`.
- **`PATCH /tasks/:id/complete`**, added strictly test-first: the five-behavior paper spec became
  five tests first — red transcript (5/5 failing with `Route PATCH … not found`) captured
  **before** a single line of implementation existed, then handler+service, then green; nothing
  in the file was written after it.
- **CI**: `.github/workflows/ci.yml` at the repo root running `npm ci → prisma generate →
  typecheck → test` on Node 24 with `working-directory: modules/m08-zod-validation`, plus the
  status badge in the module README.
- **One contract fix**: malformed JSON used to answer **500 `INTERNAL_ERROR`** (body-parser's
  `SyntaxError` fell through to the programmer-error branch, echoing the parser message). The
  central handler now maps `type: "entity.parse.failed"` to the frozen **400 validation
  envelope** (`"Body must be valid JSON"`, no details — the body never parsed far enough to have
  a field path). Discovered by probe during the build, flagged as out-of-scope, owner chose the fix.

## Why it's built this way (key decisions)

- **Why this test-database strategy — what breaks without cleanup.** The suite's whole premise
  is that tests run against *the* database, so tests also run against *each other's* rows: without
  `beforeEach` truncation, test 1's created task makes test 5's `total: 3` a `total: 4`, a
  duplicate-title 409 fires on a leftover instead of the arrange, and list-order expectations
  become positional lies. `--randomize` then converts every such dependency from "latent" to
  "failing on some run". Cleanup must be **FK-safe** (`task_tags` before the tables it references;
  `task_tags.tag` is `Restrict`, so deleting tags first is a hard error, not a warning), and the
  pinned `DEFAULT_OWNER_ID` user must be **re-provisioned after every reset** — `tasks.user_id`
  is NOT NULL, so without that row every POST would honestly 400/P2003. Deleting `test.db`
  itself in `globalSetup` and replaying migrations means the schema is never a leftover either —
  "rebuild from recorded history" (m08's non-destructive pattern) applied to tests.
- **Toolchain fallback: NONE needed — `@swc/jest` worked on the first probe.** The brief's
  fallback order (ts-jest → TS6-for-tests → CommonJS) went untouched. What made it work is
  worth repeating: explicit `.ts` import specifiers everywhere (no extensionless resolution for
  ESM to guess), a `.mjs` Jest config (the config file itself needs no transform), native `.mjs`
  setup files (infra that configures the transform pipeline should not depend on it), and the
  generated Prisma client already emitting `.ts` with `importFileExtension = "ts"`. The probe ran
  *before* any real test — m08's "probe the composition you ship" paid off again.
- **What test-first changed about the handler.** Writing the test first forced the *contract*
  decision before the code: what does completing an already-done task do (idempotent 200, not an
  error — settled in the first red), does the endpoint own anything besides `status` (no:
  title/description/priority/dueDate untouched — pinned), which seam produces the 404 (reusing
  `mapPrismaError` rather than inventing a new path). Because the test file existed first, the
  implementation had exactly one job: make 5 named expectations true — there was no opportunity
  to design the endpoint around whatever was easiest to write. It also kept the surface minimal:
  no body → `idParamsSchema` is the whole contract → no new schema file at all.
- **Coverage thresholds as config, not aspiration.** `coverageThreshold` globs on
  `src/routes/**` and `src/services/**` turn "80%+" into a failing exit code. Instrumenting only
  our code (`collectCoverageFrom` excludes `src/generated/**`) keeps the denominator honest —
  generated client code would otherwise pad the number while proving nothing.

## How to build it (teach it to the next trainee)

### The isolated-test-database guide

1. **Point the tests at their own file first, not last.** Create `.env.test` with
   `DATABASE_URL="file:./test.db"`. Gitignore it — and then realize the consequence: a fresh CI
   checkout will not have it. So the *worker setup file* must **force** the variable
   (`process.env.DATABASE_URL = "file:./test.db"`) rather than merely dotenv-loading it. Order is
   the whole trick: Jest runs `setupFiles` *before* the test file's imports are evaluated, and
   `src/db.ts` reads `DATABASE_URL` **at module load**, and `dotenv` never overrides an existing
   variable — so whichever sets it first wins, and tests must be first. Get this wrong and the
   suite silently runs against `dev.db`, where `resetDatabase()`'s `deleteMany` would **empty your
   development data**. There should be no escape hatch: the suite always uses `test.db`.
2. **Build the schema from recorded history, once per run.** A Jest `globalSetup` file (plain
   `.mjs`, runs in the main process) deletes any `test.db` from last run and spawns
   `prisma migrate deploy` with `DATABASE_URL` forced in the child env. Replay migrations; never
   `migrate reset` (destructive, consent-gated, rewrites history — m08 pattern). Env vars set in
   `globalSetup` *do* reach workers (they inherit the main process env), but the per-worker
   `setupFiles` copy exists anyway so isolation doesn't depend on that inheritance detail.
3. **Empty the database before every test, children first.** One helper, called from
   `beforeEach`: `task_tags → tasks → tags → users`, then re-create the pinned owner row. Keep
   the seed's discipline: parents after children are gone, and idempotent re-runs.
4. **Make test data deterministic.** Factories take explicit ids and `createdAt` values
   (`sequencedDate(i)`), so `sort=createdAt&order=asc` has one right answer that cannot drift.
   Tests that assert *ordering* must never rely on insertion timing.
5. **One writer.** SQLite allows a single writer, so the script is
   `node --experimental-vm-modules node_modules/jest/bin/jest.js --runInBand` — cross-platform
   (no Unix `NODE_OPTIONS=` prefix), one worker, one `test.db`. (The alternative — one DB file
   per worker via `JEST_WORKER_ID` — buys parallelism at the cost of N schemas to maintain.)
6. **Close the connection.** `setupFilesAfterEnv` registers `afterAll(() => prisma.$disconnect())`
   once, so every suite gets it without copy-paste.
7. **Verify the isolation, don't assume it.** A one-line test asserting
   `process.env.DATABASE_URL === "file:./test.db"` inside the worker, plus a run with `.env.test`
   physically removed (the CI simulation), converts the scariest silent failure into a red test.

## Concepts worth explaining

**Why a real database instead of mocks.** A mock of the database is the test author's *belief*
about the database. Everything this module must prove only exists in the real engine: `P2002`
(the real unique constraint tripping) is the *source* of the 409; a mocked `create()` would only
throw what someone coded the mock to throw, so the suite would assert itself. Same for FK
violations (P2003), "row really is gone after DELETE" (a second, fresh query — only real storage
can attest to persistence), and FK-safe cleanup order (SQLite *refuses* the wrong order; a mock
would politely allow it). The cost of realism is exactly the isolation machinery above — and
that cost is the point: if you can't isolate real state, your tests were never proving the thing
that fails in production. Mocks still belong at the *boundaries* — the suite's only spy is
`jest.spyOn(TaskService.prototype, "getById")` forcing a programmer error to observe the 500
envelope; the DB under it stays real.

**Red → green → refactor, honestly.** Red means *failing for the intended reason*, not merely
failing: the captured transcript shows `Route PATCH … not found` — the endpoint's absence, not a
typo in the assertion. That distinction is what makes the red run evidence; a test that fails
because it's wrong teaches nothing when it flips green. Green means "the named expectations are
true", after which refactor runs under the safety net — in this build, the refactor step was
recognizing the endpoint needed *no new schema* (`idParamsSchema` suffices) precisely because the
tests had already pinned the contract. The cycle's real product isn't the code, it's the
*decision record*: each expectation was argued (in the test) before it was implemented.

## What tripped me up

- **`vi` instead of `jest`** — Vitest muscle memory: `import { vi } from "@jest/globals"` fails
  in Jest with an export error *at module load* (all other tests green, suite "failed to run").
  Cheap to fix, but ESM import errors surface as suite-load failures, which reads like a
  toolchain problem — check the import names first.
- **`await` inside a non-async callback** — I passed `() => prisma.task.create({ userId: (await …)
  })` to a helper; SWC's parser rejected the whole suite before a single test ran. The error
  pointed at the exact line, but the *category* was "suite failed to run" again — same mislead.
- **My own no-leak test defeated by the dev-mode echo.** The 500 test threw
  `new Error("…at src/very/private.ts:42")` and then asserted the body contains no `src/` —
  but non-production *by design* echoes `err.message`, so the assertion failed on its own
  fixture. Fix: path-free message, assert on what only a **stack** could carry (`.ts:` frames).
  Lesson: an anti-leak assertion must distinguish the contract (no stack, ever) from the
  intentional dev affordance (message echo when `isProduction=false`).
- **The gitignored-`.env.test` CI hole.** The DoD says `.env.test` must not be committed — which
  means CI's worker would dotenv-load *nothing* and `db.ts` would fall back to `file:./dev.db`,
  putting `resetDatabase()`'s `deleteMany` on development data. Found by walking the CI sequence
  mentally before running it; fixed by forcing the env var in `setupFiles`, then proven by running
  the whole suite with `.env.test` physically removed. Always simulate the CI checkout locally.
- **Branch coverage initially 74.35%**, failing the 80% gate: `mapPrismaError`'s P2003 arm and
  its no-context ternaries are unreachable through the tasks-only API (pinned owner). The honest
  fix was not lowering the threshold but *testing the seam directly* with **genuine** Prisma
  errors from the real client (create with a dangling FK, duplicate insert, update of a missing
  row) — m07's evidence-proofs became a permanent suite member.
- **`--randomize` exposed nothing** — because isolation was built first (full `beforeEach` reset,
  deterministic factories, no cross-test references) rather than retrofitted. Three randomized
  runs green. The lesson inverts the brief's warning: order-dependence is what you get when
  cleanup is an afterthought.

## Checkpoint evidence

- **Red → green (test-first):** `evidence/m09/tdd-red.txt` (5/5 failing, `Route PATCH … not
  found`, captured *before* implementation existed) → `evidence/m09/tdd-green.txt` (all green
  after handler+service); no test in the file was written after green — all five predate
  the code.
- **Coverage ≥ 80% on routes and services:** `evidence/m09/coverage.txt` — routes
  100/100/100/100, services 100% stmts · 84.61% branches · 100% funcs · 100% lines; thresholds
  are enforced by `jest.config.mjs` (the run exits non-zero below them, as observed before the
  seam tests were added).
- **`--randomize`:** `evidence/m09/randomize.txt` plus two further randomized runs in-session —
  58/58 each.
- **Green CI on a PR:** **[PR #15](https://github.com/juzzthynperez/node-js-training-ground/pull/15) open, CI run green** (job `test`, 5m14s, the full `npm ci →
  prisma generate → typecheck → test` sequence on Node 24):
  [run 39710797860](https://github.com/juzzthynperez/node-js-training-ground/actions/runs/39710797860). A real runner is also the strongest possible form of the
  no-`.env.test` CI simulation — a fresh checkout has no such file, and the forced
  `DATABASE_URL` held on it. Pre-PR code review before opening: `blocked: false`, 0 high
  findings (medium/low notes backlogged in the retro). Badge in the module README now
  reflects it. **Merge remains the owner's step**, and this PR stacks on the still-open
  [#14](https://github.com/juzzthynperez/node-js-training-ground/pull/14) (m08 / Task API v3) — merge #14 first and this PR's diff then
  shrinks to its two commits (mirrors m08: the box closes on merge, not on open).
- Also verified: `tsc --noEmit` green with `tests/**` in the program; `.env.test`/`test.db`
  confirmed ignored via `git check-ignore`.

## What I'd do differently

1. **Walk the CI sequence before writing tests, not after building them.** The
   gitignored-`.env.test` → `dev.db` fallback was a data-destruction bug sitting in the design the
   whole time; it took a mental CI walkthrough to see it. In the capstone, the first test I write
   will assert the *environment invariants* (DB path, NODE_ENV, worker identity) — the
   infrastructure's contract — before any behavior test exists to depend on them.
2. **Treat "suite failed to run" as its own failure class.** Both `vi` and the stray `await`
   presented identically (load-time), and both times I first suspected the toolchain. A personal
   rule: import errors and parse errors are *authoring* bugs — fix in the file, don't debug the
   pipeline.
3. **Design coverage expectations with the test plan, not after the gate fails.** The
   unreachable-arms discovery at the end cost a redesign cycle; had I mapped "which branches are
   reachable through HTTP, and which need direct-seam tests" while writing the suites, the mapping
   suite would have existed from the start.
4. **Capture evidence as I go with a named index.** Rebuilding transcripts (probe, red, green)
   would have been painful after compaction; the `evidence/m09/README.md` index made each claim
   one file away. For M10: write the index file first, then fill it.
5. **Push consent earlier, not at the end.** The build sat complete while PR mechanics wait on
   consent; in the capstone I'd ask for the push/PR consent as soon as the branch is green, so CI
   evidence accrues while the write-up is drafted.
