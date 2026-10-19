# M03 Write-Up — Node.js Runtime & npm

> Filled in as I went (Build to Teach), 2026-10-19, from the actual runs.
> Node v24.21.0 via `fnm exec --using=24` — local default is still v22; see "What tripped me up".

## What I built

Two programs, both merged to `main` through PRs in `node-js-training-ground`:

1. **`log-report` CLI** ([PR #6](https://github.com/juzzthynperez/node-js-training-ground/pull/6)) —
   streams a synthetic ~100 MB newline-delimited access log, parses each record line by line,
   aggregates status codes / methods / top paths, and writes a JSON report. Options come from
   `node:util`'s `parseArgs` (`--file`, `--out`, `--top`, `--help`). The run stats are printed
   at exit, including the peak RSS the process ever used — so the "did it stream?" question
   answers itself.
2. **Bare `node:http` server** ([PR #7](https://github.com/juzzthynperez/node-js-training-ground/pull/7)) —
   three JSON endpoints (`GET /health`, `GET /items`, `POST /items`) with routing, body
   parsing, status codes, and error shaping all written by hand. No framework anywhere.

The trainer-provided `generate-log.js` fixture (not part of the lab — generating data isn't
the exercise) produces the access-log file: 100.0 MB, 746,041 records.

## Why it's built this way (key decisions)

- **Streaming approach: `createReadStream` + `readline` interface, one record at a time.**
  The whole aggregation state is three `Map`s keyed by status/path/method — a few kilobytes —
  no matter how big the file gets. Evidence on the full file: 746,041 lines parsed, peak RSS
  **59.6 MB**. The RSS number is self-reported by the CLI (`process.memoryUsage.rss()` sampled
  every 25 ms plus a final check), because "check its memory behavior on the full file" is a
  DoD item and I wanted the check baked into the tool, not bolted on.
- **What would have broken if I'd loaded the file instead:** I ran the control — same
  aggregation but `readFileSync` + `split('\n')` over the same 100 MB file. Peak RSS
  **235.5 MB**, ~4× the streaming run. At 100 MB the machine survives it; the point is that
  the cost scales with the *file*, not with the *work*. A stream costs kilobytes of state; a
  whole-file load costs the file's bytes plus the split copies. Feed either version a 10 GB
  log and only one of them still runs. (Fun detail from the control: it was *faster* — 285 ms
  vs 440 ms — one big sequential read beats per-line dispatch. Streaming wins on memory, not
  latency. Honest numbers in `evidence/m03/`.)
- **Hand-rolled routing:** a dispatch table keyed by `` `${method} ${pathname}` `` — a plain
  object, one handler per key. I chose this over an `if/else` chain because adding an endpoint
  is one entry, and the 404-vs-405 distinction falls out naturally: if any key matches the
  pathname with a different method, respond 405 with an `allow` header; only a truly unknown
  path gets 404. URL parsing is `new URL(req.url, 'http://localhost')` — never string-split
  the path yourself.
- **Hand-rolled body parsing:** collect `data` chunks into an array with a 1 MB size cap
  (413 when exceeded), `Buffer.concat` on `end`, `JSON.parse` in a try/catch (400 on failure),
  then a field check (422 with field-level details — a deliberate preview of the M05/M08 error
  envelope). Every one of those steps is a thing Express gives you in one middleware call.

## How to build it (teach it to the next trainee)

A guide to streaming a large file through a CLI, from the reasoning I actually used:

1. **Name the failure you're avoiding.** Before choosing APIs, decide what "wrong" means:
   here, memory that grows with the file. Everything below is downstream of that sentence.
2. **Pick the streaming primitive, not the API du jour.** `fs.createReadStream(path)` gives
   you a Readable of chunks. Chunks are *not* lines — a line can straddle chunk boundaries —
   so don't parse chunks. Pipe the stream into `node:readline`'s `createInterface`; it
   guarantees you see complete lines and handles the boundary case for you.
3. **Keep state constant-size.** `for await (const line of lines)` — per line, update
   `Map` counters and discard the line. The line is garbage-collected the moment the loop
   iterates; the counters are the only thing that survives. That invariant *is* the proof of
   streaming: peak memory is O(state), not O(file).
4. **Make the tool self-reporting.** Sample `process.memoryUsage.rss()` on an interval
   (`unref()` the timer so it can't keep the process alive) and print peak RSS at exit. Now
   every run is also its own memory benchmark — the DoD check is free forever.
5. **Fail loudly, exit predictably.** `stat()` the file first with a clear stderr message and
   `process.exit(1)` — the OS-level ENOENT stack trace teaches nothing. Reserve exit 0 for
   "report written". Bad `--top`, unreadable file, write failure: all exit 1.
6. **Verify on the real size, with a control.** Run on the full file, then re-run the same
   aggregation through a `readFileSync` version. Two numbers side by side (59.6 vs 235.5 MB)
   are a demonstration; one number is a claim.

## Concepts worth explaining

**Streams (and why they exist).** Node streams exist because the alternative — "read the
whole thing, then work" — breaks down exactly when data gets interesting: big files, network
sockets, anything arriving over time. A stream turns "a big problem" into "the same small
problem, many times": one line of the log is a tiny, complete unit of work, and processing it
needs the same kilobytes whether the file has ten lines or ten million. The subtlety beginners
hit: streams are *asynchronous and chunked*, so chunk ≠ line ≠ record — that's why
`readline` sits between the raw stream and my parser. If you remember one sentence: a stream
lets you pay for the data you've *used*, not the data that *exists*.

**The event loop and the thread pool (single-threaded, but not single-context).** JavaScript
in Node runs on one thread executing one call stack — that's why a busy synchronous loop
freezes everything, including the server we just wrote. But "single-threaded" describes the
JS, not the whole process: libuv keeps a thread pool (4 threads by default) for the blocking
parts — file I/O, `crypto` heavy work, DNS — and the *event loop* is the scheduler that
decides which JS callback runs next when results come back. `createReadStream` doesn't block
that one thread while 100 MB streams past; it schedules many small reads, and the loop
interleaves them with everything else. M02's async patterns (`await`, promise coordination)
are the same mechanism one level up — in M03 I finally got to see what those `await`s were
waiting *for*.

## What tripped me up

- **The version manager doesn't do what the config file promises.** `.nvmrc` pins 24; a fresh
  shell still answered `v22.23.2` — fnm doesn't auto-apply the pin here. Same M01 lesson,
  third costume: every command in this module ran through `fnm exec --using=24 -- node …`
  after probing `node --version` first. A checklist item is done when the *tool answered*.
- **The bare server's pain was quieter than expected — which is the lesson.** Nothing
  *crashed* while hand-rolling routing and body parsing; the pain is the ~100 lines of code
  that merely get you to parity with `app.get('/items', handler)`: the dispatch table, the
  chunk accumulation, the size cap, the 400-vs-422 distinction, the `allow` header, the
  JSON error envelopes I had to invent. Express removes exactly that — plus the bugs I didn't
  write today but would have next week (half-consumed bodies, encoding, timeouts, headers
  sent twice).
- **Windows can't deliver my own signals to my own server.** The SIGINT/SIGTERM graceful
  shutdown is wired (`server.close()` drain + 5 s force-exit guard), but from this harness
  neither delivery method works: cross-process `process.kill(pid, 'SIGINT')` was silently
  ignored (handler never ran, server kept listening), an in-process
  `process.kill(process.pid, 'SIGINT')` terminated the process with the default action
  instead of running the handler, and `taskkill /F` just hard-kills. Evidence and full
  detail in `evidence/m03/signal-handling-notes.md`. The write-up claim stays honest:
  handler wired and reviewed; interactive Ctrl+C is the manual verification path.
- **The control was faster than the CLI.** I expected streaming to win everywhere; it wins on
  memory and loses on latency (285 vs 440 ms). Worth knowing: streaming is a *memory*
  strategy. Line-splitting overhead on 746k lines is real; if memory is unlimited and the
  file is bounded, a single read can win. Node's own guidance picks streams for big/unknown
  sizes — which the DoD's "genuinely streams" requires anyway.

## Checkpoint evidence

Full captures in `node-js-training-ground/evidence/m03/` (trainer-private, `INDEX.md` is the map):

- **Both programs merged to `main` by PR:** [PR #6](https://github.com/juzzthynperez/node-js-training-ground/pull/6)
  (CLI, merge `fa6bb04`) and [PR #7](https://github.com/juzzthynperez/node-js-training-ground/pull/7)
  (server, merge `d17a3e7`).
- **CLI memory behavior on the full ~100 MB file:** `cli-run-full.txt` —
  `Parsed 746041/746041 lines from 100.0 MB in 440 ms (peak RSS 59.6 MB)`;
  control run (`cli-run-control.txt`): `235.5 MB` peak on the same file. The JSON report
  itself (`data/report-full.json`) carries a `streaming.peakRssMb` field.
- **Bare server's three endpoints responding:** `endpoints.txt` — live HTTP captures for
  GET /health (200), GET /items (200), POST /items (201), plus 400/422/405 (`allow` header)/404.

## What I'd do differently

- **Sample memory from the first millisecond.** I started the RSS sampler after argument
  parsing; it happened to be fine (startup allocations are tiny), but a self-reporting
  benchmark should own the whole measurement window.
- **Test the graceful-shutdown path where signals actually work.** A POSIX environment (or an
  interactive terminal) would have let me verify SIGINT end-to-end instead of wiring-and-
  reviewing. Next time I'd structure the server so the handler logic is a function I can
  unit-drive with a fake `server` object, independent of OS signal delivery.
- **Cap the paths map in the CLI from the start.** With a fixed synthetic path list it stayed
  at 21 keys, but on a real log with unbounded distinct paths (query strings!), `pathCounts`
  grows without limit — the streaming invariant I brag about breaks at the aggregation layer.
  A top-N sketch or a bounded map belongs in v2.
- **Write the PR body as I built, not after.** Both PR descriptions were reconstructed from
  evidence files at the end; writing them during the build would have forced the explanations
  while the details were live — exactly the Build-to-Teach point this module keeps proving.
