# M03 Tasks — Node.js Runtime & npm

Explain what Node.js is and how it runs code, use core modules, and manage dependencies safely
with npm.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; skim your m02 write-up (you'll reuse those async
      patterns).
- [x] Get the synthetic ~100MB log file from your trainer (generator script or download link)
      before starting the CLI lab.
- [x] Continue the branch → PR → merge habit for this module's Friday-gate
      submission.

## Research (do before or alongside the build, not after)

- [x] What Node is: V8 + libuv + core APIs, the single-threaded event loop and thread pool,
      blocking vs non-blocking, when Node is (and isn't) a good fit.
- [x] Modules: ESM vs CommonJS, `"type": "module"`, `import.meta.dirname`/`import.meta.filename`,
      the `node:` prefix.
- [x] Core APIs you'll need: `node:fs/promises`, `node:path`, `node:os`, `node:events`,
      `node:stream` (why streams exist), `node:crypto` (`randomUUID`), `node:timers/promises`,
      `node:util` (`parseArgs`, `promisify`).
- [x] `node:http` basics — enough to write a bare server with manual routing and body parsing.
- [x] `process`: `argv`, `env`, exit codes, and the `SIGINT`/`SIGTERM` signals.
- [x] Config and ergonomics: environment variables, `.env`, `node --env-file`, `node --watch`,
      `node --run`.
- [x] Globals: `fetch`, `URL`, `AbortController`, `structuredClone`.
- [x] npm: `package.json` anatomy, semver ranges (`^`/`~`), the lockfile, `install` vs `ci`,
      `update`, `outdated`, scripts, `npx`, dependencies vs devDependencies, `engines`, `npm audit`.
- [x] Supply-chain hygiene: how to vet a package before installing, review lockfile diffs, and
      understand lifecycle scripts.
- [x] Debugging: `console` methods, `node --inspect` with the VS Code debugger, reading stack
      traces.
- [x] Note (awareness only): `node:test` and `node:sqlite` exist — this course uses Jest and Prisma
      instead, so you don't need to use either.

## Build

- [x] Build the `log-report` CLI: stream the log file (don't load it fully into memory), aggregate
      counts, accept options via `parseArgs`.
- [x] Have the CLI write its aggregated counts out as a JSON report.
- [x] Build the bare `node:http` server with three JSON endpoints, writing the routing and body
      parsing by hand (no framework).

## Verify

- [x] Both programs merged to `main` through a pull request.
- [x] The CLI genuinely streams the file — check its memory behavior on the full ~100MB file, not
      just correctness on a tiny sample.
- [x] You can explain, specifically, what a framework like Express would remove pain from, based on
      what hurt while writing the bare `node:http` server.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built": the CLI and the bare server, as you finish each.
- [x] "Why it's built this way (key decisions)": why this streaming approach specifically, what
      would break if you'd loaded the file fully into memory instead, and how you designed the
      hand-rolled routing/body parsing.
- [x] "How to build it (teach it to the next trainee)": write the streaming-CLI guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up" (especially what made the bare server painful — that's the point of
      this lab).
- [x] "Checkpoint evidence": both programs merged to `main` by PR, the CLI's memory behavior on the
      full log file, and the bare server's three endpoints responding.
- [x] Close out "What I'd do differently".
