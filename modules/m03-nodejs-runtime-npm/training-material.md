# M03 — Node.js Runtime & npm

**Week 1 · Days 4–5 · 6 hours**

## 1. Getting Started

### What you'll learn and build

In M03 you learn what Node.js actually is when it runs your code, how to reach its core APIs
without a framework, and how to manage dependencies with npm so your projects stay safe and
reproducible. By the end you will have built two working programs:

1. **`log-report`** — a command-line tool that streams a ~100 MB log file and writes a JSON
   report of aggregated counts, without ever holding the file in memory.
2. **A bare HTTP server** — three JSON endpoints built directly on `node:http`, with routing
   and body parsing written by hand.

### Connecting M03 to the async patterns learned in M02

M02 gave you the async vocabulary: promises, `async`/`await`, `Promise.all`, and the mental
model of the event loop's queues. M03 shows you what those `await`s were *waiting for*. When
you write `for await (const line of lines)` in this module's lab, the event loop you predicted
output-order for in M02 is the very thing scheduling that loop's next iteration. The
coordination skills don't change here — only the collaborator: instead of `fetch` and timers,
you coordinate with files, streams, and sockets.

### Understanding the two deliverables: `log-report` CLI and bare HTTP server

The two deliverables are deliberately uncomfortable. The CLI forces you to confront what a
100 MB file does to memory if you are careless. The bare server forces you to confront every
small job a web framework normally does for you: routing, body parsing, status codes, error
shapes. You will feel the friction — that is the point. Module 5 introduces Express, and this
module's pain is what will make you *appreciate* it instead of just being told about it.

### Setting up the project in your existing `node-js-training-ground` repository

Your practical work lives in the shared `node-js-training-ground` repo, one folder per module.
Every command below runs from **the repository root** of that repo unless a command says
otherwise.

First, check your runtime — this module targets Node 24:

```bash
node --version        # should print v24.x
```

If it prints something older, your shell isn't applying the repo's `.nvmrc` pin. Use your
version manager (M01) to switch — for example `fnm use 24` — and re-check. Every `node`
command in this module assumes v24.

Start the branch for your first deliverable and create the module folder:

```bash
git switch main
git pull
git switch -c feat/module-03-log-report
mkdir -p modules/m03-nodejs-runtime-npm
```

Each deliverable gets **its own branch, its own PR, and its own merge**. You are on the CLI's
branch now; the server gets a second branch after the CLI is merged (Section 7 shows exactly
where).

Inside `modules/m03-nodejs-runtime-npm/`, create a `package.json` that marks the folder's
code as ES modules:

```json
{
  "name": "m03-nodejs-runtime-npm",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24" },
  "scripts": {
    "log-report": "node log-report.js",
    "server": "node server.js"
  }
}
```

You will fill in `log-report.js` and `server.js` as you work through this module; the
`package.json` is here from the start because it changes how Node interprets every `.js`
file around it (Section 3).

### Obtaining the synthetic log file from your trainer

The CLI lab streams a synthetic access-log file of roughly 100 MB — newline-delimited, one
record per line, like:

```text
203.0.113.42 - - [19/Oct/2026:10:15:07 +0000] "GET /api/users?page=3 HTTP/1.1" 200 4821
```

**Your trainer supplies this file** — ask for it (or its download link / generator output)
**before** you start Section 6. Generating practice data this large is not part of the lab.

Put the file where git will never see it: the repo already ignores `evidence/`, so this
module uses

```text
evidence/m03/data/access.log
```

Create that folder and save the file there (`mkdir -p evidence/m03/data`). Never commit the
data file — it is practice data, not source. Every CLI command in this module uses that path.

## 2. Understanding the Node.js Runtime

### What Node.js is: V8, libuv, and core APIs

Node.js is three things bolted together:

- **V8** — Google's JavaScript engine (the same one in Chrome). It compiles your JS to
  machine code and provides the language itself: the types, the operators, the garbage
  collector.
- **libuv** — the C library that gives Node its superpowers: the event loop, the thread pool,
  filesystem access, networking, timers. When your JS talks to the operating system, it talks
  through libuv.
- **Core APIs** — the JavaScript-facing modules you import with the `node:` prefix:
  `node:fs`, `node:path`, `node:http`, and the rest. They are the bridge between V8's pure
  JavaScript world and libuv's operating-system world.

Your code runs inside V8; anything that touches disk, network, or time goes through libuv;
the core APIs are the vocabulary you use to ask for it.

### The event loop, thread pool, and single-threaded JavaScript execution

Node executes your JavaScript on **one thread**: one call stack, one piece of code running at
a time. When that piece of code finishes, the **event loop** picks the next callback to run —
from queues fed by completed I/O, timers, and promise resolutions.

Some operations (file reads, DNS lookups, heavy `crypto`) can't be done by a callback — they
need the operating system to block. That's the **thread pool** (libuv's pool of four worker
threads by default): your single JS thread hands the blocking work to a pool thread, and when
that thread finishes, the result is queued for the event loop to deliver back to your code as
a callback (which is what your `await` was waiting on).

Three consequences you can state precisely:

- JS is single-threaded → a long synchronous loop freezes *everything* on that thread.
- I/O is not single-threaded → thousands of concurrent file/network operations are fine,
  because the pool and the OS do the waiting.
- Callbacks never *interrupt* your code → the ordering questions you practiced in M02
  (`setTimeout` vs promise) are the event loop choosing between queues, and now you know
  what's under the hood of those queues.

### Blocking vs non-blocking operations

**Blocking** means the JS thread sits and waits before running the next line. **Non-blocking**
means the request is handed off and your code moves on, receiving the result via callback,
promise, or event.

```js
// BLOCKING — nothing else on this thread runs while the file is read:
const data = readFileSync('huge.log', 'utf8');

// NON-BLOCKING — the read is handed to libuv; the loop stays responsive:
const stream = createReadStream('huge.log');
// ...the data arrives in pieces while other work can run in between
```

The blocking version is simpler to write and fine for tiny files. The non-blocking version is
what you reach for when "simple to write" would cost "stops the world" — especially inside a
server, where one blocking read stalls every other connected client. Section 6's lab is your
chance to measure this trade-off on a real 100 MB file.

### When Node.js is—and isn't—a good fit

Node shines when work is **I/O-bound**: servers handling many concurrent slow clients, CLIs
doing file and network work, proxies, scrapers, anything event-driven. Its single thread means
one process serves thousands of idle-but-connected clients cheaply.

Node is a poor fit when work is **CPU-bound**: image processing, video encoding, heavy crypto,
big computations. Those occupy the one JS thread and starve everything else. Also a poor fit
for apps that need a different language runtime's mature library ecosystem. If the job is
mostly *waiting*, Node is excellent. If the job is mostly *computing*, beware.

### Understanding how asynchronous patterns from M02 apply to Node.js programs

Nothing about M02's patterns changes; only their sources. `fetch` calls become file reads and
socket accepts. `Promise.all` over API calls becomes "handle these connections concurrently".
A concrete preview of your CLI's core loop:

```js
for await (const line of lines) {
  // 'lines' emits a new line whenever one is ready from disk —
  // the same "wait, then continue" contract as await fetch(...)
}
```

The failure patterns transfer identically: an I/O callback that throws is a crash unless you
catch it; a floating promise from an un-awaited write is a bug you'll chase at 2 a.m. The
discipline M02 built — every async failure arrives the same way, every promise is awaited or
returned — is this module's through-line.

## 3. Modules and Node.js Core APIs

### ES modules vs CommonJS

Node supports two module systems. **CommonJS** is the original: `require()` calls,
`module.exports`, files loaded synchronously. **ES modules (ESM)** are the JavaScript
standard: `import`/`export`, evaluated asynchronously, understood by browsers and every
modern tool.

```js
// CommonJS
const { parseArgs } = require('node:util');
module.exports = { helper };

// ES modules (what this course uses)
import { parseArgs } from 'node:util';
export { helper };
```

The industry has standardized on ESM, and this whole course uses it. You'll still *read*
CommonJS in older packages and stack traces — knowing it exists is enough for now.

### Configuring `"type": "module"`

Node decides how to interpret `.js` files from the nearest `package.json`. That's why
Section 1 had you create one with:

```json
{ "type": "module" }
```

Now every `.js` file in that folder is treated as an ES module, and `import` statements work
without complaints. The alternative — renaming files to `.mjs` — works too but is unnecessary
when a whole project is ESM. (`.cjs` is the mirror escape hatch for a CommonJS file inside
an ESM project.)

> **Checkpoint:** run `node --version` from the module folder, then a one-line
> `node -e "console.log('ok')"` to confirm your shell and runtime are the ones you think
> they are. Wrong-runtime bugs are the most expensive kind to find later.

### `import.meta.dirname`, `import.meta.filename`, and the `node:` prefix

Inside an ES module, `import.meta` describes *this file*:

```js
import.meta.filename  // absolute path to this file: /home/you/project/server.js
import.meta.dirname   // absolute path to this folder:  /home/you/project
import.meta.url       // the file:// URL of this file
```

You'll need `import.meta.dirname` in the CLI lab to resolve the log file and report paths
relative to the script instead of the shell's current directory.

The **`node:` prefix** tells Node "this is a core module, not an npm package":

```js
import { readFile } from 'node:fs/promises'; // core module — always available
import express from 'express';                // npm package — needs node_modules
```

Always use the prefix: it's unambiguous, it documents intent, and it prevents an npm package
named like a core module from silently shadowing one.

### Working with files and paths using `node:fs/promises` and `node:path`

`node:fs/promises` is the promise-based filesystem API — use it with `await`:

```js
import { readFile, writeFile, stat } from 'node:fs/promises';

const stats = await stat('package.json');
console.log(stats.size, stats.isFile());
await writeFile('out.json', JSON.stringify({ ok: true }, null, 2));
```

`node:path` manipulates path *strings* — never hand-concatenate paths with `/`:

```js
import { join, dirname, basename, extname } from 'node:path';

join('modules', 'm03', 'log-report.js')  // joins with the OS separator: '/' on macOS/Linux, '\' on Windows
basename('/a/b/report.json')              // 'report.json'
extname('/a/b/report.json')               // '.json'
dirname('/a/b/report.json')               // '/a/b'
```

In the CLI lab you'll use `stat` to fail early on an unreadable log file and `join` to build
the report's output path.

### Exploring system information with `node:os`

`node:os` describes the machine you're running on — useful for CLIs that adapt to their host.

> **Reference note:** `platform()` returns `'win32'`, `'darwin'`, or `'linux'`; `cpus().length`
> is the core count; `totalmem()` is RAM in bytes; `hostname()` is the machine name. Reach for
> these when a program needs to behave differently per machine — not needed for the labs.

### Understanding events and streams with `node:events` and `node:stream`

`node:events` is Node's observer pattern — any object can emit and listen for named events:

```js
import { EventEmitter } from 'node:events';
const bus = new EventEmitter();
bus.on('ready', (n) => console.log('ready', n));
bus.emit('ready', 1);
```

`node:stream` builds on that idea: a **Readable** stream emits `'data'` events (chunks of
bytes) and `'end'` when done. Streams exist so that *big data* never has to exist *whole in
memory* — you process it in pieces. Three facts to carry into the lab:

- A stream gives you **chunks**, not lines. A line can straddle two chunks; naïve chunk
  parsing will corrupt records that happen to cross the boundary.
- `createReadStream('file')` is the everyday entry point.
- `for await (const chunk of stream)` awaits the stream directly — events, promisified.

The line-boundary problem is real but has a standard fix, which Section 6 walks you through.

### Generating identifiers with `node:crypto` and `randomUUID`

`node:crypto` is Node's core cryptography module. You need one thing from it this module:

```js
import { randomUUID } from 'node:crypto';

randomUUID(); // '27e2f6c1-5b3a-4c2e-9e1d-...' — a unique ID for items, requests, anything
```

The bare-server lab uses it to give each created item a unique `id`.

### Using `node:timers/promises` and `node:util`

`node:timers/promises` gives you promise-flavored timers — the building blocks of your M02
`sleep` utility, now first-party:

```js
import { setTimeout as sleep } from 'node:timers/promises';
await sleep(150); // awaitable pause — no wrapper needed
```

`node:util` carries two tools you'll use constantly:

```js
import { parseArgs, promisify } from 'node:util';

// parseArgs: real command-line argument parsing, no npm dependency
const { values } = parseArgs({
  options: {
    file: { type: 'string', short: 'f' },
    top: { type: 'string', default: '10' },
    help: { type: 'boolean', short: 'h', default: false },
  },
});
// values.file → 'report.log' (just the value) or undefined; defaults already applied
```

`parseArgs` is the backbone of the CLI lab — you'll design its options in Section 6.

> **Reference note:** `promisify` converts a callback-style function into a promise-returning
> one (`const fn = promisify(oldFn)`). You rarely need it with modern core APIs (they're
> promise-based already), but you'll see it in older code and libraries.

### Awareness only: `node:test` and `node:sqlite`

Two core modules exist that this course deliberately doesn't use:

- **`node:test`** — Node's built-in test runner. Real enough, but this program standardized on
  **Jest** (M09) for its ecosystem and tooling.
- **`node:sqlite`** — a built-in SQLite binding. This program uses **Prisma + SQLite**
  (M06/M07) instead.

You don't need to learn either. Just know that "Node can't do X natively" is often wrong —
check before adding a dependency, which is the next section's entire theme.

## 4. Managing Processes and Configuration

### Using `process.argv`, `process.env`, and exit codes

`process` is Node's handle on the running program itself.

**`process.argv`** — the command-line words, raw:

```bash
node log-report.js --file access.log --top 5
```

```js
process.argv;
// [ '/path/to/node',          // [0] the node executable
//   '/path/to/log-report.js',  // [1] the script
//   '--file', 'access.log',    // [2+] your arguments, raw ]
```

You *could* slice and scan this array by hand — and in the next lab you'll see why you
shouldn't: `parseArgs` from Section 3 does it correctly (short flags, `--flag=value`,
defaults, errors).

**`process.env`** — the environment variables as an object of strings. Everything is a
string; everything is configuration-from-outside:

```js
const port = Number(process.env.PORT ?? 3000);
```

**Exit codes** — how a program tells the shell (and CI, and scripts) whether it succeeded:

```js
process.exit(0); // success
process.exit(1); // failure — bad input, crashed operation
```

Convention: `0` = success, anything non-zero = a failure class. Scripts and CI gate on this,
so your CLI must *exit* non-zero on failure, not just print an error.

### Handling `SIGINT` and `SIGTERM`

Signals are operating-system messages to a running process. You need two:

- **`SIGINT`** — interrupt (Ctrl+C in a terminal).
- **`SIGTERM`** — terminate (the polite kill; what `kill <pid>` and orchestrators send).

By default both kill the process instantly — mid-write, mid-response, mid-cleanup. Handlers
let you shut down *gracefully*:

```js
process.on('SIGINT', () => {
  console.log('shutting down');
  server.close(() => process.exit(0)); // stop accepting, finish in-flight work
});
```

Wire this into the bare server. One caveat you'll hit on Windows: programmatic signal
delivery (sending SIGINT to another process from code) doesn't reliably invoke these handlers
there — interactive Ctrl+C is the honest test. Wire the handler, and verify it in a real
terminal.

### Environment variables and `.env` files

Environment variables keep configuration (ports, paths, secrets) *out of your code*. For
development, they're collected in a `.env` file:

```bash
# .env — never committed; .gitignore it
PORT=3000
LOG_FILE=evidence/m03/data/access.log
```

`.env` is a plain file of `KEY=value` lines. It is **not** read automatically by Node —
that's the next tool.

### Loading configuration with `node --env-file`

```bash
node --env-file=.env server.js
```

Node reads the `.env` file and populates `process.env` before your code runs — no npm
package required (`dotenv` exists for older setups and for loading at runtime; the flag
covers the common case). The result in your program is the same either way:
`process.env.PORT` just works.

### Automatically restarting programs with `node --watch`

```bash
node --watch server.js
```

Edit a file → Node restarts the process. That's it. For a server with in-memory state this
matters constantly (your `items` array resets — watch it surprise you once, then remember).
Combine flags freely: `node --watch --env-file=.env server.js`.

### Running package scripts with `node --run`

Since Node 22, `package.json` scripts run without npm. The scripts you declared in Section 1:

```bash
node --run log-report    # runs "node log-report.js"
node --run server        # runs "node server.js"
```

It reads `scripts` from `package.json` and runs them. Handy when you want a reproducible
command without pulling the whole npm CLI into the loop. (npm's own `npm run` still works —
`node --run` is the lightweight native sibling.)

### Using built-in globals: `fetch`, `URL`, `AbortController`, and `structuredClone`

Four browser-standard globals live on Node's global object — no imports, no packages:

```js
// fetch — HTTP client, promise-based (you used it in M02)
const res = await fetch('https://api.example.com/items');

// URL — parse URLs properly; never string-split a URL
const url = new URL('/search?q=node', 'http://localhost:3000');
url.searchParams.get('q'); // 'node'
url.pathname;              // '/search'

// AbortController — cancellation, same as M02
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 5000);
await fetch(url, { signal: ctrl.signal });

// structuredClone — deep-copy with zero dependencies
const copy = structuredClone({ when: new Date(), nested: { arr: [1, 2] } });
```

The `URL` class earns its keep in the server lab: it's how you'll parse incoming request
paths by hand (`new URL(req.url, 'http://localhost')`), without ever splitting a string.

## 5. Managing Dependencies Safely with npm

### Understanding `package.json`

`package.json` is the manifest of a Node project — identity, dependencies, and behavior in
one file. You already wrote one in Section 1; here is what each part means:

```json
{
  "name": "m03-nodejs-runtime-npm",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24" },
  "scripts": { "log-report": "node log-report.js" },
  "dependencies": { "some-lib": "^1.4.0" },
  "devDependencies": { "jest": "^30.0.0" }
}
```

- `name`/`version` — identity (matters if published; hygiene matters regardless).
- `private: true` — refuses accidental `npm publish`. Use it for apps.
- `type` — the module-system switch from Section 3.
- `scripts` — the commands `node --run` (and `npm run`) execute.
- `engines` — which runtime versions the project supports; documentation until you enforce it.
- `dependencies` / `devDependencies` — next topic.

### Dependencies vs devDependencies

- **`dependencies`** — needed for the program to *run in production*. Runtime libraries.
- **`devDependencies`** — needed only to *develop*: test frameworks, linters, build tools.

`npm install <pkg>` adds to `dependencies`; `npm install --save-dev <pkg>` adds to
`devDependencies`. The distinction is not pedantic: production installs
(`npm ci --omit=dev`) ship *only* dependencies — so anything in the wrong bucket either
bloats production or breaks the build.

### Semantic version ranges: `^` and `~`

Versions are `MAJOR.MINOR.PATCH` (semver): breaking changes bump MAJOR, features bump MINOR,
fixes bump PATCH. Ranges in `package.json` say how far updates may auto-advance:

```json
"^1.4.0"  // >=1.4.0 <2.0.0 — MINOR and PATCH updates (safe additions)
"~1.4.0"  // >=1.4.0 <1.5.0 — PATCH updates only (most conservative)
"1.4.0"   // exactly 1.4.0 (rarely useful in package.json; the lockfile pins exactly anyway)
```

`^` is the default and correct for libraries you trust; `~` when you've been burned by a
minor release. The lockfile — next — is what makes either choice *reproducible*.

### Understanding the package lockfile

`package-lock.json` records the **exact** resolved version (and integrity hash) of every
package in the entire dependency tree. `npm install` writes it; committing it means every
teammate and CI run installs the identical tree. Two rules:

1. **Always commit the lockfile** for applications.
2. **Never hand-edit it** — it's generated, and its diffs are your audit trail (see
   supply-chain hygiene below).

The lockfile is why `^1.4.0` is safe: the *range* allows updates, the *lockfile* freezes the
actual tree until you deliberately update.

### `npm install` vs `npm ci`

- **`npm install`** — resolves `package.json` ranges, may update the lockfile, installs.
  The everyday development command.
- **`npm ci`** — installs **exactly** what the lockfile says, from scratch: deletes
  `node_modules`, ignores ranges, fails if lockfile and manifest disagree. The CI/release
  command — reproducible by construction.

Rule of thumb: develop with `install`, ship and test releases with `ci`.

### Managing updates with `npm update` and `npm outdated`

```bash
npm outdated          # what's installed vs latest allowed vs latest available
npm update            # update within the package.json ranges (rewrites the lockfile)
npm install lodash@4 # deliberately move a range (lodash 4.x is the current major)
```

The workflow that keeps you safe: `outdated` to see, `update` to move within range, check the
lockfile's diff, run tests, commit. Update the manifest (`package.json`) only when you intend
to cross a boundary (e.g. a MAJOR).

> **Try it:** your M03 folder has no dependencies yet, so `npm outdated` there is empty —
> run it anyway to see the tool's output shape. In M05's Task API (which does depend on
> Express) this same command becomes part of your weekly routine.

### Defining and running npm scripts

Scripts are documented, shareable commands — the CLI's user manual for your teammates:

```json
{ "scripts": {
  "start": "node server.js",
  "dev": "node --watch --env-file=.env server.js"
} }
```

```bash
npm run dev        # or: node --run dev
```

`npm`/`node --run` also prepend `node_modules/.bin` to `PATH`, so scripts can call installed
tools by name (`jest`, `prisma`) without paths.

### Using `npx` and the `engines` field

**`npx`** runs a package's CLI **without installing it into your project**:

```bash
npx cowsay "it works"
```

First run downloads to a cache and executes; subsequent runs reuse it. Ideal for
"run this tool once" commands — and for CLIs of your own packages (like `prisma` in M07).

**`engines`** declares runtime expectations:

```json
{ "engines": { "node": ">=24" } }
```

By default this is documentation. `engine-strict=true` in `.npmrc` makes installs *fail* on
mismatches — worth knowing; the course pins versions centrally via `.nvmrc` instead.

### Checking vulnerabilities with `npm audit`

```bash
npm audit             # list advisories in your dependency tree
npm audit fix         # apply semver-compatible fixes (rewrites the lockfile)
```

`npm audit` cross-checks your lockfile's exact versions against the npm advisory database and
tells you what's vulnerable and how far the fix moves you. It's a starting point, not a
shield: it only sees *known* vulnerabilities in *installed* packages — which is why the next
topic exists.

### Package supply-chain hygiene: vetting packages, reviewing lockfile changes, and understanding lifecycle scripts

Real-world npm safety is three habits:

1. **Vet before installing.** Before `npm install anything`:
   - Does it have recent commits, real users (downloads), an open repo, and an answer for
     its issue tracker?
   - Does its *size* match its job? A one-function utility with 12 dependencies is a red
     flag — every transitive dependency is someone else's code you now execute.
   - Prefer packages you've seen recommended in trustworthy contexts (this course's stack
     table is one such context).
2. **Review lockfile diffs.** The lockfile diff after any install/update tells you the
   *truth* of what changed: new packages appearing (especially transitive ones you didn't
   ask for), integrity hashes, version jumps. "I installed a formatter" must not silently
   become "37 new packages".
3. **Understand lifecycle scripts.** npm packages can run code automatically on install:
   `preinstall`/`install`/`postinstall`. That's how native modules build — and how a
   malicious package executes *the moment you install it*, before you ever import it.
   `npm ci --ignore-scripts` disables them when auditing; know that installing *is* running
   code.

The theme: `npm install` is not "downloading a library", it's "executing someone else's
code inside your project". Treat it with the review you'd give a merge request.

## 6. Building the `log-report` CLI

### Understanding the synthetic newline-delimited log file

Your input is a ~100 MB access log supplied by your trainer, one record per line (see
Section 1 for where yours lives):

```text
203.0.113.42 - - [19/Oct/2026:10:15:07 +0000] "GET /api/users?page=3 HTTP/1.1" 200 4821
```

Read one carefully: client IP, three dashes, a bracketed timestamp, a quoted
`"METHOD path PROTO"`, a status code, a response size. Your CLI will summarize the whole
file: how many requests, per status code, per method, and the top paths — written out as a
JSON report.

Open `modules/m03-nodejs-runtime-npm/log-report.js` — create the file if it doesn't exist
yet — and work through the steps below.

### Designing a command-line utility with `node:util`'s `parseArgs`

Real CLIs take options. Design yours with `parseArgs` (Section 3):

- `--file` / `-f` (string) — path to the log file. Required.
- `--out` / `-o` (string) — where to write the report. Suggest a default of
  `log-report.json`.
- `--top` (string) — how many paths to include. Suggest a default of `10`.
- `--help` / `-h` (boolean) — print usage.

Design rules that pay off immediately:

- **Validate before working.** No `--file` → print usage, `process.exit(1)`. `--help` →
  usage, `process.exit(0)`. Bad `--top` (not a positive integer) → reject.
- **Strings arrive as strings.** `--top 10` gives you `'10'` — convert and re-validate
  (`Number.isInteger`) yourself; `parseArgs` guarantees the *shape*, not the *sense*.
- **Usage text is a feature.** One line, both exit paths documented, no guessing.

Test this layer *before* touching the file: run with `--help`, with a missing `--file`,
with `--top banana`. Each should behave sanely and exit with the right code.

### Reading a large file with streams instead of loading it entirely into memory

The heart of the lab. Two APIs form one pipeline:

```js
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const stream = createReadStream(file, { encoding: 'utf8' });
const lines = createInterface({ input: stream, crlfDelay: Infinity });

for await (const line of lines) {
  // one complete line at a time
}
```

Why this shape:

- `createReadStream` emits **chunks** of bytes — not lines. A 64 KB chunk can end mid-line,
  and the line's remainder arrives in the next chunk. Parsing chunks directly corrupts
  whatever record straddles the boundary.
- `createInterface` (from `node:readline`) fixes exactly that: it buffers across chunks and
  emits **complete lines**. `crlfDelay: Infinity` makes it handle Windows line endings
  without splitting on the `\r`.
- `for await` consumes the async iterator of lines — chunk boundaries handled; your loop
  just sees a sequence of strings.

The memory invariant to *state out loud*: your aggregation state (some `Map`s) is measured
in kilobytes and does not grow with the file. That's what "genuinely streams" means, and
the verification at the end of this section is where you prove it.

### Processing and aggregating log records line by line

Inside the loop, parse each line with a regular expression matching the record shape. Start
from this pattern and adjust it against real lines from your file:

```js
const LOG_PATTERN = /^(\S+) \S+ \S+ \[([^\]]+)\] "(\w+) (\S+) [^"]*" (\d{3}) (\d+|-)/;
//   groups:        IP     — —  timestamp      method path       status  bytes
```

Two disciplines from this shape:

- **Count the malformed, don't crash on them.** Real logs contain junk. A `malformed`
  counter turns junk into information; a thrown exception turns it into an outage.
- **Name the groups.** The regex captures by position — destructure into named variables so
  the code reads like the log line.

### Counting relevant information, such as status codes and request paths

Maps keyed by the thing you're counting:

```js
statusCounts.set(status, (statusCounts.get(status) ?? 0) + 1);
```

Do the same for methods and paths, and keep a running total of bytes served (watch for the
`-` placeholder the format allows instead of a number). These `Map`s are your entire memory
footprint — a few kilabytes even on the 100 MB file. (One honest caveat to keep in mind:
with a synthetic log whose paths come from a fixed list, `pathCounts` stays tiny; on a real
log with unique query strings it could grow without limit. Knowing where your invariant can
break is part of the design.)

### Writing the aggregated results to a JSON report

When the stream ends, assemble one plain object and write it with `writeFile`:

- `generatedAt` — an ISO timestamp.
- `source` — file path, file size (from `stat`), total lines, parsed lines, malformed lines.
- `totals` — requests, bytes served, distinct paths.
- `statusCodes`, `methods` — the count maps, as plain objects, sorted by count descending.
- `topPaths` — the N most common paths (`--top`), as `{ path, count }` entries.
- `streaming` — the peak-memory figure you measure in the verification step below (add the
  field now, fill it in when the sampler exists).

Sort before serializing (counts descending reads naturally), and use two-space indentation —
the report is *for humans to inspect*.

### Handling invalid input, file errors, and command-line options

Every failure exits non-zero with a clear stderr message — no stack traces for user errors:

```js
try {
  fileStats = await stat(args.file);
} catch {
  console.error(`Cannot read log file: ${args.file}`);
  process.exit(1);
}
```

The full matrix to cover: missing/unreadable file, non-file path, invalid `--top`, unwritable
`--out` location (the write-stream error path), and interrupting mid-run (Ctrl+C — leave
partial files alone or write atomically; know which you chose).

### Verifying memory behavior against the full approximately 100 MB file

The Definition of Done says *genuinely streams, verified on the full file* — so verify like it.
You will do three things, and record **your own** numbers (yours depend on your machine and
your exact file — the numbers are evidence, not targets).

1. **Make the CLI measure itself.** Sample `process.memoryUsage().rss()` (the resident-set
   size: the RAM your process actually holds) on an interval — remember `timer.unref()` so
   the sampler can't keep the process alive — track the maximum, and print it in the run
   summary at exit. Also embed it in the JSON report's `streaming` field.
2. **Run on the full file**, not a sample, and record the output:

   ```bash
   node modules/m03-nodejs-runtime-npm/log-report.js \
     --file evidence/m03/data/access.log --out evidence/m03/data/report.json --top 5
   ```

   Your streaming peak RSS should stay far below the file's ~100 MB — it reflects your
   small aggregation state, not the input size.
3. **Run a control.** Write a second, throwaway script (keep it in `evidence/`, not in the
   module folder) that computes the same counts but loads the whole file first
   (`readFileSync` + `split('\n')`), measuring peak RSS the same way. Compare:

   - If your streaming run's peak sits well below the file size and the control's peak sits
     at or above it (it must hold the file *plus* the split copies), you've demonstrated the
     difference. Record both numbers and explain the gap in one sentence.
   - Expect the control to sometimes be *faster* — one big read beats per-line dispatch.
     Streaming is a memory strategy, not a speed strategy. If you observe that, say so
     honestly in your notes; it's a real property of the trade-off.

4. **Check correctness too** — totals sum, top paths match the log's distribution, the
   malformed count matches how many junk lines you fed it. Memory without correctness is a
   benchmark, not a working CLI.

> **Checkpoint (Definition of Done, part 1):** the CLI parses the full file without loading
> it whole, prints a peak-RSS figure well below the file size, produces a correct JSON
> report, and exits 1 with a clear message on bad input. Your recorded streaming-vs-control
> numbers are the evidence.

## 7. Building a Bare HTTP Server

### Understanding the basics of HTTP requests and responses

HTTP is a request/response protocol over TCP: the client sends a **request** (method, path,
headers, optional body), the server sends a **response** (status code, headers, body). The
verbs and codes you'll implement:

- `GET /path` — read a resource → `200` with the data.
- `POST /path` — submit data to create → `201` with the created thing (and `400`/`422` when
  the data is wrong).
- Status codes are the contract: `2xx` ok, `4xx` *the client* erred, `5xx` *you* erred.

Everything a framework abstracts — routing, parsing, serialization — is this protocol
handled by hand. Doing it once by hand is how the abstraction becomes obvious instead of
magical.

### Creating a server with `node:http`

Get one response working before designing anything:

```js
import { createServer } from 'node:http';

const server = createServer((req, res) => {
  res.writeHead(200, { 'content-type': 'application/json; charset=utf-8' });
  res.end(JSON.stringify({ hello: 'world' }));
});

server.listen(3000, () => console.log('listening on http://localhost:3000'));
```

Run it (`node modules/m03-nodejs-runtime-npm/server.js`), curl it, then Ctrl+C it. The
callback gets Node **incoming message** (`req`, a readable stream of the request) and
**server response** (`res`, a writable stream). Note what you must do *by hand* here: pick
the status, set the content type, serialize the JSON, and remember that `res.end` — not the
callback's return — is what finishes the response.

### Defining three JSON endpoints

Move the server to its own branch first — the CLI is its own deliverable:

```bash
git switch main && git pull
git switch -c feat/module-03-http-server
```

Your module folder (with `package.json`) is already on `main` if you merged the CLI branch;
if you haven't merged it yet, finish and merge the CLI first — the two deliverables are
separate PRs by design.

Scope: `GET /health` (liveness + stats), `GET /items` (list), `POST /items` (create). Back
them with an in-memory array — no database until M06/M07 — seeded with two items whose
`id` comes from `randomUUID()` (Section 3) and `createdAt` from `new Date().toISOString()`.

A helper removes the repetition you'd otherwise copy-paste three times:

```js
function sendJson(res, statusCode, payload) {
  const body = JSON.stringify(payload);
  res.writeHead(statusCode, {
    'content-type': 'application/json; charset=utf-8',
    'content-length': Buffer.byteLength(body),
  });
  res.end(body);
}
```

(`content-length` isn't strictly required — Node sets it — but setting it explicitly is good
hygiene and teaches what the header means.)

### Implementing routing manually

Routing is "map (method, path) to a handler". The simplest honest version — a dispatch
table:

```js
const routes = {
  'GET /health': (req, res) => sendJson(res, 200, { status: 'ok' }),
  'GET /items': (req, res) => { /* your handler */ },
  'POST /items': (req, res) => { /* your handler, with body parsing */ },
};

createServer((req, res) => {
  const url = new URL(req.url, 'http://localhost'); // never string-split paths
  const handler = routes[`${req.method} ${url.pathname}`];
  if (handler) return handler(req, res);
  // ...404 and 405 handling
}).listen(3000);
```

Design choices worth noticing: the key includes the **method** (GET /items and POST /items
are different endpoints); `URL` parses the path correctly (query strings excluded from
`pathname`); unknown paths get a JSON 404 *you* wrote — core `node:http` sends nothing of
its own here, so without this branch a request to an unknown path would simply hang until
the client gave up.

> **Exercise:** complete the 404/405 distinction yourself. An unknown path gets a JSON 404.
> A *known* path with the wrong method gets **405 with an `allow` header** listing the valid
> methods — how you detect "path exists, method doesn't" from the dispatch table is part of
> the design work.

### Reading and parsing request bodies

A request body arrives as a **stream of chunks** — not a ready-made object. Build a
`readJsonBody(req, res, callback)` helper that satisfies four requirements:

1. **Accumulate** the `data` chunks into an array (`req.on('data', ...)` — data events
   deliver pieces, not wholes).
2. **Cap the size** (~1 MB): past the cap, respond `413` and `req.destroy()` — a hostile
   client must not OOM you.
3. **Assemble and parse** on `end`: `Buffer.concat(chunks).toString('utf8')`, then
   `JSON.parse` inside a try/catch — malformed JSON is a `400`, not a crash.
4. **Report upward** via the callback so the POST handler can decide `201` vs `422`.

This is one endpoint's body handling — count how many lines it costs, and hold that number
until this section's last topic.

### Returning appropriate status codes and JSON responses

The status matrix for `POST /items`, each earned:

| Situation | Code | Why |
| --- | --- | --- |
| Valid body, item created | `201` | Something was created — say so, not just `200` |
| Unparseable JSON | `400` | The client sent garbage |
| Valid JSON, missing/empty `name` | `422` | Well-formed but semantically invalid — with `details: [{ path, message }]` |
| Body over the cap | `413` | Protect the server |

Require `name` to be a non-empty string; on success, push `{ id: randomUUID(), name,
createdAt }` to `items` and echo the created item in the `201` body. That `details` array is
a deliberate preview: M05's Task API (and M08's Zod integration) standardizes exactly this
field-level error shape.

### Handling invalid requests and errors

Beyond the status matrix: the 404 for unknown paths, the 405 + `allow` header for
wrong-method, and the uncaught-exception backstop (`process.on('uncaughtException')` logs and
exits non-zero — a last resort, not a design). The through-line: **every response is
deliberate**. A server that answers every request with a JSON body of known shape is the
difference between a protocol peer and a hobby script. Wire the SIGINT/SIGTERM graceful
shutdown from Section 4 as well, and verify it with a real Ctrl+C in an interactive terminal.

### Identifying the pain points that a framework like Express would remove

You just wrote the server. The final DoD item asks you to explain — specifically, from your
own code — what a framework like Express would remove pain from. Build the answer while you
build the server: keep a short friction list as you go. Prompt questions to make sure your
list is complete (answer each in your own words, pointing at your own lines):

- What would a route like `/items/:id` cost you in the dispatch table?
- How many lines is your body-parsing helper, and what would `express.json()` replace it
  with?
- Where did status codes and error JSON get assembled by hand?
- Where would you put logging or an auth check so it applies to *all* routes — and what does
  the absence of a middleware concept mean for that?
- What happens to your process if a handler throws synchronously?

Express (M05) replaces each of these with one line — and this lab is why that means something
to you now.

## 8. Debugging Node.js Applications

### Using `console` methods to inspect program behavior

`console` is four tools wearing one name:

```js
console.log('value:', x);      // general output
console.error('bad input');    // stderr — use it for errors so stdout stays clean data
console.warn('deprecated');    // stderr, labeled warn
console.dir(obj, { depth: null }); // full object expansion (log() truncates nesting)
console.time('parse');
/* ... */
console.timeEnd('parse');      // 'parse: 42ms' — cheap phase timing
console.trace('how did I get here'); // stack at this point
```

Two habits that pay immediately: route errors to `console.error` (a CLI's stdout may be
piped into another program — noise there is corruption), and `console.dir` for nested
objects where `console.log` prints `[Object]`.

### Reading stack traces and locating errors

A Node stack trace, decoded:

```text
TypeError: Cannot read properties of undefined (reading 'name')
    at formatItem (server.js:41:20)      ← where it threw — your code
    at Server.<anonymous> (server.js:66:5) ← who called it — your code
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5) ← runtime internals — usually noise
```

Read **bottom-up**: the top frame is the crash site; frames in *your* files trace the path
there. Frames inside `node:internal/...` or `node_modules` describe the machinery — skim
past them unless the error is clearly *inside* a library's contract. First error line gives
you the *what* (`TypeError` on `undefined`); frames give you the *where*. "Cannot read
properties of undefined" always means one thing: something you expected to exist didn't —
walk up the frames to the data's origin.

### Debugging with `node --inspect` and the VS Code debugger

For the bugs `console` can't reach (a wrong value computed deep inside a long loop, state
that only breaks under a specific request):

```bash
node --inspect modules/m03-nodejs-runtime-npm/server.js   # debugging protocol on :9229
```

Then attach either a browser (`chrome://inspect`) or — better for this course — **VS Code**:
Run and Debug → attach to the process (or a launch config with `"program"` and
`"runtimeExecutable": "node"`, `"args": ["--inspect"]`). The payoff capabilities:

- **Breakpoints** — pause on a line, on function entry, or on events.
- **Watch/evaluate** — inspect expressions *in the paused scope*; re-run them live.
- **Call stack** — the same stack-trace reading from the previous section, but live and
  navigable (click a frame, see that scope).
- **Conditional breakpoints** — `if (status === '500')` on a hot loop, without 700k prints.

The `--watch` flag composes with inspection (`node --watch --inspect server.js`) — restart
*and* breakpoint.

### Investigating CLI parsing, file-processing, and HTTP request failures

Failure classes and their first moves, from this module's own labs:

- **CLI parses wrong values** — print `process.argv` (or `values` after `parseArgs`) at the
  top; the bug is nearly always *before* your logic (flag shape, string-vs-number, defaults).
- **File processing yields garbage counts** — take 10 real lines, `console.log` the regex
  match against them, and compare group-by-group. Boundary bugs (straddled lines) show up as
  malformed counts that vary with file size — that variation is the tell.
- **HTTP request misbehaves** — first instrument with a `console.error` line logging the
  method and URL at the top of the handler, then `curl -i` (the `-i` shows status and
  headers — half of all HTTP bugs are visible there). Wrong JSON out: log the *serialized
  body* right before `res.end`.

### Diagnosing memory usage and asynchronous behavior

- **Memory**: `process.memoryUsage()` (object with `rss`, `heapUsed`, `heapTotal`) on an
  interval — if the line trends *upward forever* in a long-running process, you're retaining
  (a Map keyed by unbounded input, listeners re-registered per request). The CLI's peak-RSS
  sampler from Section 6 is this technique in miniature; reuse it.
- **Async behavior**: for "did my promise actually resolve / my listener fire", a single
  targeted `console.trace` at the suspect point beats any amount of staring. For ordering
  questions, you already own the tool from M02: predict the order first, then let a run
  confirm or refute — a prediction you can't lose.
- **Hangs**: Node has no built-in "print a stack on a signal" feature — `Ctrl+\` (SIGQUIT)
  isn't handled by Node and will simply kill the process on POSIX. The reliable move is to
  start the suspect program under `node --inspect` from the beginning and attach the VS Code
  debugger when you need it (`--watch --inspect` composes), or add a targeted
  `console.trace` at the await you suspect. "It hangs" usually resolves to "an await that
  never settles" — find the await that lacks a matching resolution.

## 9. Hands-On Practice and Verification

Work through these in order; each one has a definition of "done" you can check mechanically.

1. **Build the `log-report` CLI** to stream the supplied log file, aggregate counts, accept
   command-line options, and generate a JSON report. Done when: `node
   modules/m03-nodejs-runtime-npm/log-report.js --file evidence/m03/data/access.log
   --out evidence/m03/data/report.json` prints a run summary, the report contains sorted
   status-code counts, method counts, top paths, and totals, and unreadable input exits 1
   with a clear message.
2. **Verify the CLI using the full ~100 MB file and inspect its memory behavior.** Done
   when: you have recorded your own peak-RSS numbers for the streaming run *and* a
   load-entire control, the streaming figure sits far below the file size, and you can
   explain the gap between the two in one sentence. Your numbers are the evidence — record
   them; they depend on your machine.
3. **Build a bare `node:http` server** with three working JSON endpoints and hand-written
   routing and body parsing. Done when: no framework or router package appears in
   `node_modules` or `package.json`, routing is your dispatch table, and body parsing is
   your own accumulate-cap-parse code.
4. **Test all three endpoints and verify their responses.** Done when: `GET /health` → 200
   JSON; `GET /items` → 200 with the seeded list; `POST /items` with valid JSON → 201 with
   the created item; and the error paths respond correctly — invalid JSON → 400, missing
   `name` → 422 with details, wrong method → 405 with `allow`, unknown path → 404. Use
   `curl -i` and capture the headers.
5. **Debug and resolve errors encountered during both builds.** Done when: every error you
   hit is either fixed or documented with root cause — the debugging tools of Section 8
   (`console.dir`, stack-trace reading, `--inspect`, conditional breakpoints) were the
   instruments, not luck.
6. **Commit the completed programs on feature branches** in your existing
   `node-js-training-ground` repository. Done when: each program lives on its own branch
   (`feat/module-03-log-report`, `feat/module-03-http-server`), commits are scoped and
   descriptively messaged, and no data files (`*.log`, `node_modules`, generated reports)
   are in the tree.
7. **Open pull requests and merge both programs into `main`.** Done when: both PRs exist
   with bodies stating what each program does and how it was verified, and both are merged
   to `main` — that merge is this module's Friday-gate submission.

## 10. Wrap-Up and Self-Assessment

Check yourself against the module's Definition of Done — each item is a *demonstration*, not
a recognition test:

- **Explain how Node.js executes JavaScript and handles asynchronous work.** Out loud, in
  about a minute: V8 runs your JS on one thread; libuv's thread pool and the OS do blocking
  work; the event loop schedules the callbacks and promise continuations your `await`s were
  waiting on; I/O-heavy work is cheap because the waiting isn't on your thread.
- **Explain why streaming is preferable for processing large files.** The one-sentence
  version: memory cost becomes O(state), not O(file) — a few kilobytes of counters process a
  100 MB or a 10 GB file identically, while whole-file loading dies at the size of the
  input.
- **Demonstrate that the CLI generates the expected JSON report without loading the entire
  file into memory.** Run it on the full file; show the report *and* your peak-RSS line;
  show your control-run comparison and explain it in one sentence. The numbers are yours to
  record — what matters is that they demonstrate the invariant.
- **Demonstrate that all three HTTP endpoints work.** Show `curl -i` captures for the three
  happy paths and the 400/405/404 error paths — headers included.
- **Explain what Express would simplify based on your experience building a bare HTTP
  server.** Grounded in your own lines: name the code you wrote that a framework would
  replace, and what it would replace it with. If you can point at the specific functions and
  tables, you've got it.
- **Confirm both programs have been merged into `main` through pull requests.** Open the two
  merged PRs — branch, review, merge — that's the gate.
- **Review key npm dependency-management and security practices.** Recite the loop:
  vet the package before installing (size, maintenance, trust), keep ranges sensible
  (`^`), commit the lockfile, `ci` for reproducible installs, `audit` for known
  advisories — and remember the deeper truth: `npm install` *runs code* (lifecycle scripts),
  so installing is executing.

If every box above is a demonstration you can put on a screen, this module's gate is passed —
and you now know *why* M05's Express, and later `npm ci` in CI, exist. That understanding is
the deliverable that outlasts the code.
