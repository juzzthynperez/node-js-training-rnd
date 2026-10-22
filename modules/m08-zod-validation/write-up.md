# M08 Write-Up — Zod Validation & Type Inference (Task API v3)

> Filled from the actual build in `node-js-training-ground` (PR
> [#14](https://github.com/juzzthynperez/node-js-training-ground/pull/14), branch
> `feat/m08-zod-task-api-v3` from clean `main` (`5496c54`); commits `e61414a` (54 files,
> +12,540) + `887ecb8` (pre-PR review parity fixes — see the retro). Environment: Node
> **24.21.0** via `eval "$(fnm env --shell bash)" && fnm use 24` (session default was
> 22.23.2; the engine requires ≥24), `zod` **4.6.7** (`npm i zod@4`), Prisma client **7.10.1**
> generated. Evidence lives in ground `evidence/m08/` (gitignored — 8 captures + 2 runnable
> probe scripts).

## What I built

`modules/m08-zod-validation/` in the ground repo — Task API v3, every hand-written request
check deleted and replaced by Zod:

- **`src/schemas/task.schemas.ts`** — the single source of truth for the request contract:
  - `createTaskSchema` — non-strict object (unknown keys **stripped**, M07 parity), title
    trimmed then `min(1)`, `status` `z.enum(…).default("todo")`, description `optional()`.
  - `updateTaskSchema` — derived from the *same* field factory with `.partial().strict()` plus
    a cross-field `.refine()` ("at least one field"), one detail per unknown key.
  - `listQuerySchema` — a `queryParam()` pipeline per field: unknown → reject arrays
    ("Must not be repeated") → `preprocess` empty-string-means-absent → inner optional type;
    `page`/`pageSize` via `z.coerce.number()` with one custom message per field; `status`,
    `sort`, `order` as enums whose messages name every allowed value.
  - `idParamsSchema` — `z.uuid()` (accepts the seed UUIDs and the NIL UUID exactly like the
    old regex did — probe-verified).
  - The four request types (`CreateTaskInput`, `UpdateTaskInput`, `ListOptions`, `IdParams`)
    are `z.infer` aliases — the only request types in the codebase.
- **`src/schemas/env.schemas.ts`** — `envSchema` (`PORT` coerced + ranged + defaulted,
  `NODE_ENV` enum-optional) ending in a `.transform()` that emits `{ port, isProduction }`;
  `AppConfig = z.output<typeof envSchema>`.
- **`src/middleware/validate.ts`** — the typed middleware factory
  `validate({ body, params, query })`: safe-parses only the named parts, stores parsed values
  on a declaration-merged `req.valid`, forwards raw `ZodError` with `next()`. Plus
  `getValid(req, part, schema)` — the single documented boundary cast, typed by the schema you
  pass, loud (throw) if the middleware isn't mounted.
- **`src/middleware/error-handler.ts`** — new `ZodError` branch: one Zod issue → one
  `{ path, message }` detail; `unrecognized_keys` expands to one detail **per key** with the
  hardcoded M07 wording (`path: <key>`, `message: "Unknown field"`); envelope, status, and
  every other error path byte-unchanged.
- **`src/routes/tasks.ts` / `task-handlers.ts`** — `validate()` wired between router and
  handler on all five routes; handlers rewritten to read `getValid(req, part, schema)` — no
  `req.body`/`req.query` reads, no input interfaces, no try/catch.
- **`src/config.ts`** — `loadConfig()` keeps the injectable-env shape but now runs
  `envSchema.safeParse`, prints `z.prettifyError(...)`, and `process.exit(1)`s before `listen()`.
- **`src/services/task-service.ts`** — the three hand-written input interfaces deleted (types
  re-exported as the `z.infer` aliases); `create`'s data literal now carries
  `satisfies Prisma.TaskUncheckedCreateInput`, `update`'s carries
  `satisfies Prisma.TaskUpdateInput`.
- **Deleted:** `src/middleware/validation.ts` (≈170 lines of `validateCreateTask` /
  `validateUpdateTask` / `validateListQuery` / `isValidId`) — gone, not dormant.
- **Also:** `package.json` renamed (`m08-zod-validation`, new description), `.env.example`
  added (PORT/NODE_ENV/DATABASE_URL), local `.env` copied (gitignored), DB rebuilt with
  `db:deploy` + `db:seed` (4 users / 18 tasks / 7 tags).

## Why it's built this way (key decisions)

**Why this schema shape for create vs update — what did `.partial()` let me avoid repeating?**

- Both schemas are built from **one** `taskFields({ title, description })` factory + one shared
  `statusField`. The factory's only parameter is the *message wording*, because m07 froze
  different strings per context (create: `"Required, non-empty string"` /
  `"Must be a string when present"`; PATCH: `"Must be a non-empty string"` / `"Must be a
  string"`) and wire-visible strings are contract — so the *logic* is written once, the
  *wording* stays byte-identical to v2. A message-parity grep against m07's `validation.ts`
  is what surfaced this; it is now covered by `schema-sanity.txt` cases
  `u.blank_title_uses_patch_wording` / `u.bad_description_uses_patch_wording`.
- `.partial()` on the update schema means a PATCH declares no field rules of its own — add a
  field to `taskFields` and *both* endpoints learn it, with the update side automatically
  optional. `.strict()` flips create's strip-unknown to reject-unknown, which is exactly the
  v1-vs-v2 behavior difference being preserved.
- **The default must stay out of the shared base.** My first version derived update as
  `createTaskSchema.partial()` — `.partial()` over `.default("todo")` still fires the default
  for an absent key, so `PATCH {}` parsed as `{ status: "todo" }`, satisfied the
  "at least one field" refine, and answered **200 instead of 400** (caught by Bruno). The
  factory takes the bare `statusField`; only `createTaskSchema` adds `.default("todo")`.
- The empty-patch rule lands as a *whole-object* issue at path `""` — exactly where m07
  pushed `{ path: "", message: "At least one of …" }`. The pre-PR review caught that a
  `.refine()` on the strict object also fires when the object parse *failed*, so
  `PATCH {"foo":1}` collected that issue *and* the unknown-field one where m07 returned only
  the latter (its emptiness check sat behind an `errors.length > 0` early return). The rule
  now runs as a pre-parse guard (`unknown().refine(nonempty).pipe(object.partial().strict())`),
  which restores the mutual exclusivity: guard fails → only its issue; guard passes → only the
  body's own issues.

**How did you work around `req.query` being read-only in Express 5?**

- Assignment to `req.query` throws (it's a getter), and rewriting `req.valid`'s home onto
  `res.locals` would bury parsed input in an untyped bag. So: a declaration-merged
  `req.valid?: { body?; params?; query? }` that `validate()` writes after each `safeParse`,
  and handlers read through `getValid(req, part, schema)` — the schema argument *is* the
  return type (`z.output<S>`), so the cast exists in exactly one place, next to the runtime
  guarantee that the middleware just parsed this part against that schema. If the middleware
  isn't mounted, `getValid` throws instead of handing business logic `undefined`.
- One non-obvious constraint: a bare `unknown().refine().pipe(...)` field is **not** marked
  optional to Zod's object parser — every *absent* query param came back
  `expected nonoptional, received undefined` and even a queryless `GET /tasks` 400'd. The
  pipeline needs an **outer `.optional()`**; that's commented in `queryParam()` where the
  mistake was made.

**How did you keep your Zod schema and your Prisma type aligned, and what would drift if you hadn't?**

- Zod is the *request* contract (what clients may send); Prisma's generated types are the
  *persistence* contract (what the database accepts). The seam is the service layer, and the
  `satisfies` operator makes the compiler police it: `create`'s data literal is
  `satisfies Prisma.TaskUncheckedCreateInput` — **Unchecked**, not `TaskCreateInput`, because
  our data is all scalars (pinned `userId` FK + columns, no nested `user:` write) and Prisma's
  *checked* create input demands the relation object for required relations; `update`'s
  literal is `satisfies Prisma.TaskUpdateInput` (scalar-only patch, no relation keys).
- What drifts without it: the schema stops exposing a field the service still writes (or vice
  versa), and the first symptom is a runtime P2011/P2003 instead of a red squiggle. The
  deliberate *non*-alignment is documented too: `userId`/`priority` are service policy and are
  intentionally absent from the request schema — that asymmetry is why the unchecked input is
  the right target, not laziness.

## How to build it (teach it to the next trainee)

**Deriving a typed validation middleware from a single schema — the reasoning, not just the
syntax.**

1. **Start from the messages, not the types.** The envelope's `details[]` strings are frozen
   wire contract (Bruno asserts several verbatim). Write them down as constants first —
   including *both* wordings if the old code had context-specific ones — so every schema you
   build below quotes a contract value instead of inventing a sentence. Zod 4's unified
   `error` parameter takes these: `z.string({ error: MSG }).min(1, { error: MSG })`.
2. **Write each field once, in the grammar of its input.** Query strings arrive as strings
   (or *arrays* when repeated — that's your "Must not be repeated" guard, a `refine` on
   `unknown` before anything else runs), JSON body fields arrive as JSON values, route params
   arrive as path segments. `z.coerce.number()` is what turns `"2"` into `2` for the query
   schema; `z.uuid()` is the top-level format replacing the hand-written regex.
3. **Derive endpoint schemas from a bare base.** `.partial()` for PATCH, `.strict()` when the
   endpoint must reject unknown fields, `.refine()` for cross-object rules. Keep defaults out
   of the shared base — a default is a *creation* concept, and `.partial()` will still fire it.
4. **Let the schema own the types.** `export type CreateTaskInput = z.infer<typeof createTaskSchema>`
   lives in the schema file; the service imports the alias, handlers never name a type for
   input at all. Proof you can run: rename one schema field and `tsc` points at the *handler*
   that destructures it, printing the inferred shape (our capture: `id` → `taskId` →
   `TS2339: Property 'id' does not exist on type '{ taskId: string; }'` ×3).
5. **Build `validate()` as a *factory over named parts*, not a per-endpoint function.** It
   loops over `["body", "params", "query"]`, skips parts no schema was given, `safeParse`s,
   and on failure does one thing: `next(result.error)` — raw `ZodError`, no wrapping, no
   `res` usage. Returning early keeps invalid input out of every downstream layer, which is
   the whole point: the handler (and Prisma behind it) is *unreachable* for bad input.
6. **Store parsed parts where they're typed.** `req.valid` via declaration merge; handlers go
   through `getValid(req, "body", createTaskSchema)`, so the parameter schema is simultaneously
   the lookup key and the return type. Never `req.body`: it's the raw thing you just proved
   untrustworthy.
7. **Map issues to the envelope in exactly one place** — the central `error-handler` you
   already have. Shape-wise, `err.issues.flatMap` gives one `{ path: issue.path.join("."),
   message: issue.message }` per issue; the one special case is Zod's `unrecognized_keys`,
   whose `path` is the *object* path and whose payload is a `keys[]` array — expand it to one
   detail per key so `PATCH {"foo":1}` keeps m07's `{ path: "foo", message: "Unknown field" }` —
   and sorts those expanded details first, since m07 looped unknown keys before its field checks
   while Zod emits shape issues before `unrecognized_keys`.
   The envelope (`code`, `message`, `details`, `requestId`) is untouched — Zod's output is
   *mapped into* it, never leaked as-is.
8. **Wire routes as `router.post("/", validate({ body: createTaskSchema }), handler)`** — the
   middleware's position in the chain *is* the guarantee. Then delete the old validator file
   in the same change; a Zod layer *alongside* hand-written checks is two contracts drifting
   in polite agreement.

## Concepts worth explaining

**`z.infer` vs a hand-written interface (my own words).** A hand-written interface is a
*second implementation* of your validation rules — written in a different language (type
syntax vs runtime checks), at a different time, by whoever edited the schema last. Drift is
not a risk, it's a schedule: rename a field in the validator and the interface still compiles,
happily typing code that reads a property the runtime will never receive. `z.infer` removes
the second implementation: the schema is executable *and* is the type. When we renamed
`idParamsSchema`'s field to prove it, the compiler didn't complain in the schema — it complained
in `task-handlers.ts`, at the destructure, printing `{ taskId: string; }`. That error location
is the entire feature: the type's *provenance* is visible in the toolchain. Related trio:
`z.input>` is what you feed in (query strings!), `z.output`/`z.infer` is what comes out
(coerced numbers, trimmed strings) — they differ exactly when transforms sit in the schema,
which is why the middleware stores *output* and Prisma sees values that already passed through
`.trim()` and `z.coerce`.

**Why Zod is the request contract and Prisma is the persistence contract — and where they
meet.** A client may send `{"title": "  milk  ", "page": "2"}` — valid HTTP, meaningless to
the database. Prisma wants trimmed strings, enums it recognizes, and (for a required relation)
a connect, not a stray `userId`. Those are two different grammars for two different audiences,
and each layer should enforce *its own* and only its own: Zod answers "is this a well-formed
request for *this API*?" at the edge, in one `safeParse`, with field-level English in the
envelope; Prisma answers "is this a well-formed row?" at the boundary, with P-codes. The seam
is where they must *agree* — that's what `satisfies Prisma.TaskUncheckedCreateInput` buys: the
compiler checks the Zod-typed input against Prisma's grammar at the exact literal we hand over.
What must never happen is skipping the first grammar: raw `req.body` into `prisma.task.create`
means the database is your validation layer, and by then the error is a 500 or a P2011 with
no `details[]` a client can act on.

## What tripped me up

- **A field pipeline that passed every standalone test 400'd inside `z.object`.** My probe
  called `field.safeParse(undefined)` → success; the same field as an object property →
  `Invalid input: expected nonoptional, received undefined` for every *absent* key. Zod's
  object parser decides "may this key be missing?" from the field's **top-level** type, and a
  bare pipe isn't `optional`. Fix: `.optional()` on the outside of the pipeline. Lesson: probe
  composed schemas *in the composition you ship*, not just the component.
- **`.partial()` over `.default()` silently re-applied the default.** First Bruno run: `PATCH {}`
  answered 200. `createTaskSchema.partial()` kept `status: …default("todo")` live, so an empty
  patch became `{ status: "todo" }` and sailed past the "at least one field" refine. The
  default belongs to create only; both schemas now derive from a base whose `status` has none.
- **Zod 4.6.7 crashed on the object form of the `error` param in `.refine()`** —
  `{ error: { message: "…" } }` threw `iss.inst?._zod.def?.error is not a function` inside
  `finalizeIssue` when the error was read. The **string** form (`{ error: "…" }`) and the
  legacy `{ message: "…" }` both work everywhere; every schema in this build uses the string
  form. Evidence: `zod4-api-check.txt`.
- **`.strict()`'s unknown-key issue has `path: []` + `keys: ["foo"]`, not `path: ["foo"]`.**
  Mapped naively you'd emit `{ path: "", message: 'Unrecognized key: "foo"' }` — one wrong
  detail at the wrong path. The handler expands `unrecognized_keys` per key instead (and
  hardcodes `"Unknown field"`, since a schema-level `error` on the object hijacks that issue's
  message). Wire output verified by `PATCH {"foo":1}` in `400-envelope.txt`.
- **Prisma's AI consent guard blocked `migrate reset`** (7.10.x: an agent must not run
  destructive resets without explicit user consent). The target was a freshly-created, empty
  dev `dev.db`, so the non-destructive path did the same job: `npm run db:deploy` (applies the
  two recorded migrations) + `npm run db:seed`.
- **Node 22 by default, engine ≥24.** Every npm/tsx invocation needed
  `eval "$(fnm env --shell bash)" && fnm use 24 && …` in this session; the very first
  `npm install` silently ran on 22 and failed building `better-sqlite3` (prebuild download
  timed out, no VS toolchain for the node-gyp fallback). Re-running under 24 pulled the right
  prebuild.
- **The first full Bruno run was 6/23** — five list requests 400'ing (the bare-pipe bug) plus
  the empty-patch 200 (the default-leak). Both root causes were in schema *composition*, both
  were invisible to schema-level unit probes — the Bruno suite is what caught them, which is
  exactly the argument for running the frozen collection against every behavioral change.

## Checkpoint evidence

All under ground `evidence/m08/` (gitignored; files referenced by name):

| DoD | Proof |
|---|---|
| No manual `if (!body.title)`-style checks remain | `validation-removal-grep.txt` — gates [1] `if (!body.` [2] old validator names/calls [3] hand-written input interfaces [4] `req.body`/`req.query` outside comments [5] raw request objects passed to the service: **all exit=1 (zero matches)**; [6] the only module-wide `isValidId` hits are two comments (schema header, seed comment). `src/middleware/validation.ts` is deleted in commit `e61414a`. |
| Invalid input never reaches Prisma | Validation sits between router and handler on all five routes (`src/routes/tasks.ts` — `validate(…)` is the middleware arg), so the handler *cannot* run on invalid input; observed live in `bruno-run.txt` (every bad-input request is a 400 at the middleware) and at schema level in `schema-sanity.txt` (26/26 cases behave as the old validators did). |
| 400s carry the exact M05 envelope `details` shape | `400-envelope.txt` — seven live responses: missing title (`path: "title"`, `"Required, non-empty string"`), the invalid trio (`status`+`sort`+`page` with frozen messages), `PATCH {"foo":1}` (`{path:"foo","Unknown field"}` — a single detail, m07's early-return parity; the pre-PR review caught a spurious second `"At least one of …"` detail an object `.refine()` emitted after strict rejection, now fixed), `PATCH {}` (`details[0]` includes `"At least one"`), blank PATCH title, bad id (`path: "id"`), and mixed `PATCH {"foo":1,"title":123}` (unknown-field detail *first* — m07's ordering), plus `"Request validation failed"` / `VALIDATION_ERROR` on every one. |
| `tsc --noEmit` clean, handler types from `z.infer` | `typecheck.txt` (clean run) + `typecheck-inference-proof.txt` — renaming `idParamsSchema.id` → `taskId` produced three `TS2339` errors **in `task-handlers.ts`** printing the inferred type `{ taskId: string; }`, then reverted to clean. |
| Env validation fails fast | `env-fail-fast.txt` — `PORT=abc npm start` → `✖ PORT must be an integer between 1 and 65535 → at PORT`, **exit=1** before `listen()`; `NODE_ENV=staging npm start` → enum message, **exit=1**. |
| Full suite green, no regressions | `bruno-run.txt` — frozen M05/M07 collection: **23/23 requests, 30/30 tests, 21/21 assertions, PASS**. |
| Frozen-message parity with v1/v2 | `schema-sanity.txt` + message-parity grep against m07's `validation.ts` (only differences were the two PATCH wordings, now reproduced verbatim); Zod 4.6.7 API assumptions settled in `zod4-probe.ts` / `zod4-api-check.txt`. |
| v3 merged by pull request | **Pending — owner's action.** Branch `feat/m08-zod-task-api-v3` committed locally as `e61414a` from clean `main` (`5496c54`); push/PR/merge remain with the owner by convention. |

## What I'd do differently

- **Run the message-parity diff *first*, not last.** I built the shared field vocabulary
  before diffing m07's literal strings, which briefly unified two context-specific messages
  the wire had frozen. A ten-line grep of the old validator's literals at schema-design time
  turns this from a late fix into a spec input.
- **Probe compositions, not just components.** Both live bugs (bare pipe in an object,
  partial-over-default) passed standalone schema probes and died in the Bruno suite. The
  probe file should include "absent key inside `z.object`" and "empty object through
  `.partial()`" as standing cases — those are the shapes real requests produce.
- **Give the Bruno suite a one-command runner that owns the server.** `npm run test:bruno`
  assumes something is already listening on :3000; every module so far has hand-started a
  background server and hand-killed a PID. A `test:e2e` script (boot → wait for `listening` →
  `bru run` → teardown) would make the DoD check one command instead of four.
- **Move the `unrecognized_keys` wording decision into the schema layer.** The handler
  hardcodes `"Unknown field"` per key — correct, but it's message policy living in mapping
  code. A function-form `error` param on the update schema (if/when this Zod build stabilizes
  it — see the object-form crash above) would keep every wire string in `task.schemas.ts`.
- **`getValid`'s guard throws a plain `Error` (→ 500).** Right classification (wiring bug, not
  client error), but an `AppError` subclass with a clearer message would make the failure
  self-describing in the envelope instead of a generic internal error.

