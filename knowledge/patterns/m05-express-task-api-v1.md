# Patterns — discovered during M05 (2026-10-20)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Design the validator's output to be the wire format, not a shape to translate later

- `validateCreateTask` returns `{ input }` or `{ errors: ErrorDetail[] }` where
  `ErrorDetail = { path, message }` — deliberately the same as the fixed envelope's
  `details`. The handler's only job on failure is
  `throw new BadRequestError(msg, errors)`; the envelope comes out already correct.
- The alternative — validators returning `{ field, reason }` or plain strings — forces a
  translation step *inside the error path*, which is the worst possible moment to
  translate (you're mid-response, the shape is half-built, and the mapping is easy to
  get subtly wrong). Matching the wire format removes a whole class of bug.
- Standing pattern for M08: when Zod replaces these validators, the *output contract*
  (`{ path, message }[]`) must survive the swap unchanged, because the envelope is
  program-wide. Design the seam so only the engine changes, not the shape.

## The operational/programmer split is a trust boundary, not a status-code table

- `err instanceof AppError` in the central handler is the line between "trust this
  error's message" and "distrust it entirely." AppError → status/code/message are safe
  by construction. Not AppError → log the full stack server-side, respond with a fixed
  500 that leaks nothing but a requestId.
- The tempting-but-wrong fix ("just make every error an AppError") launders bugs into
  400s and hides the fact that something broke. The split's value is that a 500 in the
  logs *means* a bug — if bugs can wear 400 costumes, that signal is destroyed.
- `NODE_ENV === "production"` only tunes whether an *unexpected* error's message is
  echoed in dev. It never applies to AppError — an operational message is always safe.

## Middleware order is invariants the compiler cannot check — write them as comments in app.ts

- Three orderings are load-bearing and none is enforced by types: (1) request-ID before
  logging, because logging *reads* `req.requestId`; (2) body parser before routes,
  because handlers read `req.body`; (3) error handler last and after the 404 handler,
  because Express only delivers an error to error middleware registered *after* the
  failure point.
- Get any of these wrong and it still compiles and still runs — the bug is behavioral
  (a log line with no id; a body that's undefined; an error that falls through to
  Express's default HTML page). So the fix is documentation *in the code*: the
  `buildApp` function carries a numbered comment naming each invariant next to its
  `app.use`.
- Teaching form: order is the part of Express that looks arbitrary and is actually the
  whole program. Trainees should be able to explain *why* the error handler is last, not
  just that it is.

## Express 5 async-forwarding removes a boilerplate reflex — and the trap is muscle memory

- No handler in M05 needs `try/catch`: a `throw` in an async handler becomes a rejected
  promise that Express 5 forwards to the central error handler automatically. Writing
  handlers, the reflex to add `try { } catch (next) }` (Express 4 muscle memory) kept
  firing and had to be consciously suppressed.
- The trap worth teaching: Express 5 forwards *rejections*. A truly synchronous throw in
  a non-async handler is still caught (Express wraps handler invocation), so *both* work
  — but mixing the two styles in one file makes the error path hard to reason about.
  Keeping every handler `async` (even when the store is sync) makes the path uniform and
  makes M09's Prisma swap signature-compatible.

## Bruno CLI: three format traps that cost a full re-run

- **`tests {}` blocks run Chai, not Jest.** First headless run: all 13 declarative
  `assert` blocks passed, 18/19 script tests failed with
  `Invalid Chai property: toBe`. Fix was mechanical (`to.equal`, `.to.be.a("string")`,
  `.to.be.undefined`, `.to.include`) but cost a run. Bruno's own examples are Chai-style;
  the internet is Jest-style; the mismatch is silent until it isn't.
- **`bru run` must execute *at* the collection root** (the folder holding `bruno.json`).
  Passing the folder as an argument errors with "You can run only at the root of a
  collection." So the npm script is `cd bruno && bru run --env local`, not
  `bru run bruno`.
- **A post-response script is `script:post-response {}`, not `script {}`** — plain
  `script` runs *before* the request, so `res.getBody()` is undefined there. And an
  environment `.bru` accepts `vars` but rejects an `auth` block outright.

## Windows SIGTERM: the M03 finding now has a 4th delivery path, all confirmed dead

- The M03 retro parked a decision for M05 ("POSIX run or code-review-scoped claim").
  M05 exercised the 4th path — spawn the server as a child, call `child.kill("SIGTERM")`
  — and got the same result as M03's three: `code=null`, no `shutdown` log, handler
  skipped. libuv terminates unconditionally on Windows for programmatic signals.
- POSIX fallbacks checked before conceding: WSL's docker-desktop distro has no Node
  (exit 127); Docker Desktop's engine pipe wasn't running. No honest end-to-end run
  available in this harness.
- Scoped claim accepted: the handler is wired and code-reviewed; the signal *path* is
  not verifiable here. Repro kept at `scripts/graceful-shutdown-test.ts`, and its exit-1
  is the *expected* outcome — documented as such so a future reader doesn't "fix" it.
- Pattern generalization (from M03, now confirmed again): verify the *mechanism*, not
  the API's existence. "The handler is registered" is not "the handler ran."

## npm now silently skips postinstall scripts — a fresh-clone trap for trainees

- `npm install` left esbuild's (tsx's engine) postinstall unapproved; tsx failed to
  resolve its binary until `npm install-scripts approve esbuild`. Same for protobufjs
  (a Bruno CLI dep). Two prompts, two approvals.
- Any trainee hitting "tsx: command not found" or a missing native binary on a *fresh
  clone* should check `npm install-scripts ls` first — it's the invisible step now.
  Worth one line in whatever setup notes M06 ships.

## A teaching example that feeds a test must state the same rule as the test (2026-10-20 revision)

Section 4's request-ID example always generated a fresh UUID; section 8's Bruno test
asserted the client's inbound `x-request-id` comes back echoed. The example would have
failed the very test written three sections later — caught only by a human consistency
read, because nothing executes material against its own checkpoints. Mechanical
countermeasure: for every acceptance assertion, grep the example that feeds it for the
same behavioral keyword (`echo`/`inbound`/`randomUUID`) and require one shared rule
sentence to be the single source both sections quote. One rule statement, referenced
twice — never two renderings.

## Two renderings of the same concept is a contradiction bug class — state rules once, reference them (2026-10-20)

Section 5 originally showed the error handler twice: snippet A sent `err.message` for
*all* errors (leaking bug internals), snippet B (production-safety) forbade exactly
that — a beginner copying A silently shipped the leak B existed to prevent. Fix shape:
convert the concept to numbered rules (the four-rule handler contract), attach tiny
fragments only for the tricky mechanics (conditional `details`), and make every later
section *point at* the rules rather than re-rendering the code. Post-edit grep gate:
`message: err\.message` and `req\.requestId = randomUUID` must both return 0 in the
material — each of those strings identifies one half of one of these contradictions.
