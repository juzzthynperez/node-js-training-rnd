# Patterns — discovered during M09 (2026-10-22)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## A gitignored config file must never be load-bearing — force the variable, don't dotenv-load it

- The DoD required `.env.test` gitignored (CI-verifiable), but the worker setup only *dotenv-loaded*
  it — so on a fresh CI checkout dotenv injects nothing, `src/db.ts` falls back to its
  `file:./dev.db` default, and `resetDatabase()`'s `deleteMany` would have emptied **development**
  data while the suite "ran green" locally (dev.db exists, .env exists). The fix forces the value
  where the order already guarantees precedence — `setupFiles` runs before any test-module import,
  `db.ts` reads `DATABASE_URL` at module load, dotenv never overrides an existing key:
  `process.env.DATABASE_URL = "file:./test.db"` unconditionally, no fallback, no escape hatch.
- Generalize: a config file you deliberately don't commit is *absent by design* on every clean
  checkout — any pipeline step that silently depends on it has a missing-input bug waiting for
  CI. Two habits catch it: make critical env vars **forced in code** with the file merely
  overriding for humans, and **run the CI sequence locally with the file physically removed**
  (the simulation found it in one shot).

## Anti-leak assertions must separate the contract from the intentional dev affordance

- The 500-no-stack test threw `new Error("…at src/very/private.ts:42")` and then asserted the
  body contains no `src/` — but the handler echoes `err.message` in non-production **by design**,
  so the assertion failed on its own fixture, not on a leak. Fix: path-free fixture message +
  assert only what a *stack* could carry (no `stack` key, no `.ts:` frames, no `src/`).
- Generalize: "never expose X" tests need a fixture that *cannot legitimately contain X* —
  otherwise the test fails on the sanctioned channel and trains you to weaken it. Enumerate the
  sanctioned channels first (here: dev message echo, requestId), then assert the rest.

## The frozen suite is a floor for STATUS CODES too, not just message strings — probe error classes no test pins

- M08's contract-fidelity lesson was about message literals; M09 found the sibling gap:
  malformed JSON answered **500 INTERNAL_ERROR** (body-parser's SyntaxError fell through to the
  programmer-error branch, echoing the parser message) because *no suite in M05–M08 ever asserted
  a status for it*. The probe (`send('{"title": ')` → log status) surfaced the live behavior in
  one test; the owner chose the fix (map `entity.parse.failed` → frozen 400 envelope) over pinning.
- Generalize: for every error *class* the app can emit (parser failures, payload-too-large,
  timeouts, unsupported media type), either a test pins its envelope or an ADR pins its
  deferral — "no assertion" silently ships whatever the fall-through branch happens to do.

## A coverage gate is a forcing function — unreachable arms get real-error seam tests, not a lower threshold

- Branch coverage landed at 74.35%: `mapPrismaError`'s P2003 arm and its no-context ternaries are
  unreachable through the tasks-only API (pinned owner). Rather than exempt the file, the fix
  provokes **genuine** Prisma errors from the **real** client (dangling-FK insert, duplicate
  insert, update of a missing row, passthrough of a plain Error) — m07's one-off evidence scripts
  became a permanent suite member at 84.61% branches.
- Generalize: an uncovered branch is a fork in behavior someone once cared about. The three
  exits are: reach it through the public surface, prove it with a direct-seam test on real
  inputs, or delete it — a threshold exception is the only exit that adds no knowledge.

## "Suite failed to run" is a different failure CLASS — authoring bugs, not behavior signals

- Both M09 stumbles presented as load-time suite failures, not red tests: `import { vi }` (Vitest
  muscle memory — Jest's object is `jest`) and `await` inside a non-async arrow passed to a
  helper (SWC parser rejection). Both mislead toward "the toolchain is broken" — the exact
  reflex this program keeps retraining (M07: remembered behavior is a hypothesis).
- Generalize: triage by phase — *failed to load* = check import names and syntax in the test file
  first; *failed assertion* = check behavior. The phase the error arrives in names the file you
  must open.

## Test-first pays in CONTRACT decisions, not just in tests — the red run argued the spec

- The `PATCH /complete` red (captured before any implementation) forced five contract decisions
  that a post-hoc test would have rubber-stamped: idempotence = 200 (not 409), endpoint owns only
  `status`, 404 through the existing `mapPrismaError` seam, params-only (no new schema file —
  `idParamsSchema` suffices), registration can't collide with `PATCH /:id` (one segment vs two).
  The implementation had exactly one job: make five named expectations true.
- Generalize: the red run's value is that it names the failure for which the code is the *answer*.
  Keep the intended-reason discipline — `Route … not found` is a good red; a typo'd assertion is
  a wrong red and must be fixed before it can ever mean anything green.

## After touching a shared layer, re-run the PREVIOUS cohort's floor suite — your new tests can't prove the old contract

- The M09 work changed `error-handler.ts` (the one file every error response flows through) and
  added a route + service method — while 58 new tests all passed. But the new tests pin the
  *v4* contract; not one of them could prove the *frozen M05–M08* wire behavior didn't regress
  (they were written after the change, so they encode whatever the code now does). The check
  that could: `npm run test:bruno` against a live server — 23/23 requests, 30/30 tests green,
  including requests that predate every line of this cycle.
- Generalize: each cohort's suite only guards behavior *that cohort* saw. The validation step for
  a shared-layer change is running **every older suite you have** (Bruno, m01–m08 scripts) as
  regression evidence, because they are the only assertions written before the change existed.
  New tests prove the future; old tests protect the past; only a green run of both is a PASS.
