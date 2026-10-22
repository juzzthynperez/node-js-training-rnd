# M09 Tasks — Integration Testing with Jest + Supertest (Task API v4)

Write reliable, isolated integration tests that exercise the real Express app and a real SQLite
database.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; have your M08 write-up (Task API v3) on hand — you're
      testing it and shipping it as v4.
- [x] Set up Jest 30, Supertest 7, and `@swc/jest` in the project.
- [x] Create a separate `test.db` via `.env.test`, and confirm the transform works in ESM mode
      before writing any real tests — if `@swc/jest` doesn't cooperate with the Prisma-generated
      client, apply the brief's fallback order exactly, and record which one you needed.
- [ ] Continue the branch → PR → merge habit for this Friday-gate submission.

## Research

- [x] Testing strategy: the pyramid, unit vs integration vs end-to-end, what to test in an API, and
      why integration tests here use a real database rather than mocks.
- [x] Jest 30 basics: `describe`/`it`/`expect` from `@jest/globals`, matchers, async tests,
      `beforeAll`/`afterAll`/`beforeEach`, `test.each`, `toMatchObject`/`expect.objectContaining`,
      `rejects`/`toThrow`, and Arrange-Act-Assert.
- [x] Supertest: `request(app)` against the exported app, `.get/.post/.patch/.delete`,
      `.send`/`.set`/`.query`, status/header/body assertions, async/await style, and agents for
      cookies.
- [x] Test database strategy: a Jest `globalSetup` that runs `prisma migrate deploy` against
      `test.db`, cleaning tables in `beforeEach` in foreign-key-safe order, `--runInBand` (or one DB
      file per worker via `JEST_WORKER_ID`), factories/seed helpers, and
      `afterAll(() => prisma.$disconnect())`.
- [x] Coverage (`--coverage`, thresholds), what causes flaky tests (shared state, time, ordering),
      mocking only at boundaries (`jest.fn`, `jest.spyOn`), and why `jest.mock` hoisting doesn't
      apply in ESM (`jest.unstable_mockModule`).
- [x] The GitHub Actions workflow shape: `npm ci` → `prisma generate` → `typecheck` → `test`, plus a
      status badge.

## Build

- [x] Write tests for the happy paths across every endpoint.
- [x] Write tests for validation failures (400 with `details`), 404s, and 409s on unique
      violations.
- [x] Write tests for malformed JSON, and for pagination/filter/sort behavior on the list endpoint.
- [x] Write tests for middleware behavior: the request-ID header, the 404 handler, and that the
      error handler never leaks a stack trace.
- [x] Add `PATCH /tasks/:id/complete` test-first: write a failing test, make it pass, then
      refactor.
- [x] Set up the GitHub Actions workflow (`npm ci` → `prisma generate` → `typecheck` → `test`) and
      add the status badge to the README.

## Verify

- [x] At least 25 integration tests exist, covering all endpoints and every error path.
- [x] Green CI on a pull request.
- [x] 80% or more coverage on routes and services.
- [x] The suite passes with `--randomize` — if it doesn't, fix the ordering dependency, don't just
      avoid the flag.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": why this test-database strategy and what would
      break without cleanup, which toolchain fallback you actually needed (if any), and what
      test-first `PATCH /tasks/:id/complete` changed about how you wrote the handler.
- [x] "How to build it (teach it to the next trainee)": write the isolated-test-database guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": flaky tests, ESM/Jest friction, any ordering dependencies the
      `--randomize` run exposed.
- [x] "Checkpoint evidence": green CI on your PR, the suite passing with `--randomize`, your
      coverage report meeting 80%+, and `PATCH /tasks/:id/complete` added test-first.
- [x] Close out "What I'd do differently" — write this one especially carefully, since the M10
      capstone has no reference material except what you write here.
