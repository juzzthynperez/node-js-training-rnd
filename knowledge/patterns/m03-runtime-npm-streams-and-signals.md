# Patterns — discovered during M03 (2026-10-19)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Make the deliverable prove its own non-functional requirement

- The DoD said "genuinely streams — check memory behavior on the full file". Rather than a
  one-off measurement, the CLI samples `process.memoryUsage.rss()` (25 ms interval,
  `unref()`d so the timer can't hold the process open) and prints peak RSS on every exit,
  plus embeds it in the JSON report. The verification becomes a permanent property of the
  tool; every future run re-proves the claim for free.
- Pair with a deliberately-broken control (same aggregation, `readFileSync` + `split`): the
  contrast (59.6 MB vs 235.5 MB on the same 100 MB file) is a demonstration; a single
  number is only a claim. Keep the control out of the repo (evidence/ only) so the
  deliverable can't be confused with the anti-pattern.
- Honest note that survived review: the control was *faster* (285 vs 440 ms). Streaming is a
  memory strategy, not a speed strategy — saying so protects the teaching point from a
  trainee who benchmarks and "catches" you.

## Windows harness cannot deliver signals to your own Node process — verified, not assumed

- Three delivery paths tried for the server's SIGINT/SIGTERM graceful shutdown:
  cross-process `process.kill(pid, 'SIGINT')` (handler never fired, server kept running),
  in-process `process.kill(process.pid, 'SIGINT')` (process died with default action,
  exit 1, handler skipped), and `taskkill /F` (hard kill by definition).
- Conclusion recorded in evidence/m03/signal-handling-notes.md: on this Windows setup,
  programmatic SIGINT does not run JS handlers; interactive Ctrl+C is the only honest test.
  The write-up claim is scoped to "handler wired and reviewed" — the M02 rule again: a
  checklist item is done only when the tool *answered*.
- Teaching form: platform semantics differ beneath identical APIs; verify the *mechanism*,
  not the API's existence.

## Hand-rolled HTTP pain inventory (the M03→M05 bridge)

- The bare-server lab's learning value is the *inventory* of forgotten conveniences, so
  capture it while it's fresh: dispatch-table routing that knows no path params, the
  accumulate/cap/assemble/try-catch body function, per-call-site status codes and JSON
  envelopes, 404-vs-405 distinction (+`allow` header), no middleware seam for cross-cutting
  concerns, and one thrown handler error killing the process. Each line maps 1:1 to an
  Express feature in M05 — keep this list and reuse it as M05's opening motivation.
- The 400-vs-422 distinction (unparseable vs semantically invalid) with field-level
  `details` deliberately previews the M05/M08 error envelope — plant it early, reference it
  later.

## Dispatch-table routing + URL parsing beats string surgery

- `` `${method} ${pathname}` `` object keys: adding an endpoint is one entry; 405 falls out
  by checking whether any key shares the pathname; `new URL(req.url, 'http://localhost')`
  gives a correctly-parsed pathname (query strings excluded) with zero string splitting.
  Same shape scaled into the M05 Express router mentally.

## fnm trap: third consecutive module

- Fresh shell answered v22 while `.nvmrc` pins 24 (ground repo). Every M03 command ran
  through `fnm exec --using=24 -- node …` after a `node --version` probe. This is now a
  standing session-opener for M04+, not a one-off.

## Generator-as-fixture pattern (data too big to commit)

- The ~100MB log can't be committed (and shouldn't be — README says evidence/ is private
  and data isn't source). Resolution: commit the *generator* (small, reviewable, deterministic
  intent) under the module folder; generate the data into gitignored `evidence/m03/data/` at
  lab start. Trainees get reproducible data without a 100MB repo artifact.

## Trainee-facing revision pass: sweep, don't reread

- Revising a module doc from trainer draft to trainee material is a *sweep* problem, not a
  rewrite problem. Invariants first: heading lint (approved `##`/`###` skeletons match the
  owner outline exactly), even fence count, APPEND-MARKER=0, step ranges per section — then
  targeted greps for the things that must be gone: process vocabulary ("Build to Teach",
  "write-up", "trainer"), and *reference-build artifacts* (exact benchmark numbers, commit
  SHAs, "was run for real" claims). Zero hits is the exit criterion; no full reread needed.
- Replace every reproduced number with an instruction to record the trainee's own ("your
  streaming peak should sit far below the file size; record it") — a number in trainee
  material silently becomes a target trainees try to hit and a lie on other hardware.
- Solution-manual drift check: for each lab section, count complete implementations vs
  requirements + starting fragments + "done when" criteria. The fix is mechanical — turn
  finished code into a numbered requirements list and keep only the non-obvious API shapes
  (stream pipeline, dispatch table) as shown code.
- Consistency greps catch the accumulated small lies: every `git switch` line (one branch
  per deliverable, created once), one log-file path reused by every command, working
  directory stated once and honored.
