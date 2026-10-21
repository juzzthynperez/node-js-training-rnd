# M08 Tasks — Zod Validation & Type Inference (Task API v3)

Validate all untrusted input at runtime with Zod 4, and derive TypeScript types from the same
schemas.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; have your M05 (error envelope + validation) and M07
      (Prisma types) write-ups on hand.
- [x] Install `zod@4`.
- [ ] Continue the branch → PR → merge habit for this Friday-gate submission.
      *(branch `feat/m08-zod-task-api-v3` committed locally as `e61414a`; push/PR/merge are the
      owner's — see write-up "Checkpoint evidence")*

## Research

- [x] Why runtime validation matters: TypeScript types vanish at runtime, and `req.body` is
      untrusted data.
- [x] Zod 4 basics: primitives, the top-level string formats (`z.email()`, `z.uuid()`, `z.url()`),
      `z.object`/`z.strictObject`, arrays, enums and literals, unions and discriminated unions,
      optional/nullable/default, and `z.coerce.number()` for query strings and params.
- [x] Parsing: `parse` vs `safeParse`, and error formatting with `z.flattenError`, `z.treeifyError`,
      `z.prettifyError`, plus custom messages via the `error` parameter.
- [x] Refinements and transforms: `.refine`, cross-field rules, `.transform`, pipes, trimming/
      normalising input.
- [x] Type inference: `z.infer`, `z.input`, `z.output` — and deriving create/update/query schemas
      with `.pick`, `.omit`, `.partial`, `.extend` instead of writing them by hand.
- [x] How to work around `req.query` being read-only in Express 5 when building a typed `validate`
      middleware.
- [x] How a `ZodError` should map into the fixed M05 error envelope's `details` array.
- [x] How to keep a Zod schema aligned with its Prisma counterpart (e.g.
      `satisfies Prisma.TaskCreateInput`) without ever passing raw `req.body` to Prisma.
- [x] How to validate `process.env` once at startup in `config.ts` and fail fast.
- [x] Note (awareness only): Zod Mini and `z.toJSONSchema` exist — you don't need to use either
      here.

## Build

- [x] Write Zod schemas for create, update, list query, and params.
- [x] Build the typed `validate({ body, params, query })` middleware.
- [x] Wire the middleware into every v2 route, removing the hand-written validation from v1/v2 as
      you go — not leaving it alongside the new Zod layer.
- [x] Map `ZodError` output into the fixed M05 envelope's `details` array in your central handler.
- [x] Add env validation in `config.ts` that fails fast on startup.
- [x] Confirm, with `tsc --noEmit`, that your handler input types actually come from `z.infer`, not
      from any hand-written interface left over.

## Verify

- [ ] v3 merged by pull request. *(pending owner push/PR/merge; commit `e61414a` is local on
      the branch)*
- [x] No manual `if (!body.title)`-style checks remain anywhere in the codebase — grep for them if
      you're not sure.
- [x] Invalid input never reaches Prisma — a Zod schema rejects it first, every time.
- [x] 400 responses carry field-level `details` in the exact envelope shape fixed in M05.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": schema design per endpoint, how you handled
      `req.query` being read-only, how you kept Zod and Prisma types aligned.
- [x] "How to build it (teach it to the next trainee)": write the typed-validation-middleware
      guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up".
- [x] "Checkpoint evidence": v3 merged by PR, `tsc --noEmit` proving handler types come from
      `z.infer`, confirmation no manual checks remain, invalid input never reaching Prisma, and a
      400 response carrying `details` in the exact envelope shape fixed in M05.
- [x] Close out "What I'd do differently".
