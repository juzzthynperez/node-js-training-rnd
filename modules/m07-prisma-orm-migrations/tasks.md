# M07 Tasks — Prisma ORM & Migrations (Task API v2)

Model data with Prisma 7, evolve the schema safely with migrations, and use the typed client in the
Express API.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; have your M05 (API + error handler) and M06
      (schema + ERD) write-ups on hand.
- [x] Install `prisma@7` and `@prisma/client@7`, plus the `better-sqlite3` driver adapter — lock
      both to `^7` in `package.json` from the start.
- [x] Run `npx prisma@7 init` and confirm the datasource URL lives in the right config file for
      your Prisma version (`prisma7.config.ts`, or `prisma.config.ts` pre-7.10).
- [x] Continue the branch → PR → merge habit for this module's checkpoint.

## Research

- [x] What an ORM is, its trade-offs, and when it's still right to drop to raw SQL.
- [x] Prisma schema language: models, scalar types, `@id`, `@default`, `@unique`, `@@index`,
      `@updatedAt`, relations (1:N, implicit M:N), `onDelete`.
- [x] Migration commands: `migrate dev --name`, the anatomy of a `migration.sql` file,
      `--create-only` plus hand-editing SQL, `migrate status`, `migrate deploy`, `migrate reset`,
      and `db push` — know what each is for and when to use it.
- [x] Why you never edit an already-applied migration, what adding a required column to a table
      with existing data requires, and how migration conflicts get resolved in a team PR.
- [x] Prisma Client fundamentals: the singleton client pattern, `create`/`createMany`/`findUnique`/
      `findFirst`/`findMany`/`update`/`upsert`/`delete`, `where` with `AND`/`OR`/`NOT`/`contains`/
      `in`/`gte`, `orderBy`, pagination with `skip`/`take`, `select` vs `include`, nested writes,
      `count` and `groupBy`, `$transaction` (batch and interactive), `$queryRaw`, and query logging.
- [x] The Prisma error codes `P2002`, `P2025`, and `P2003` — what each one means.
- [x] TypeScript with Prisma: the generated model types, `Prisma.TaskCreateInput`,
      `Prisma.TaskWhereInput`, `Prisma.TaskGetPayload<{ include: … }>`, and
      `satisfies Prisma.TaskSelect`.
- [x] Seeding with an explicit `npm run db:seed` script, and inspecting data with Prisma Studio.
- [x] Concept-only: what changes with PostgreSQL, how MongoDB's document model differs, and what
      Prisma 8 changes — no lab for this, just know the shape of the answer.

## Build

- [x] Translate your M06 schema into the Prisma schema language.
- [x] Create your initial migration.
- [x] Add a column (e.g. `dueDate` or `priority`) via a second migration.
- [x] Create one hand-edited migration using `--create-only`.
- [x] Build a service layer on Prisma that replaces the in-memory store from v1.
- [x] Implement the list endpoint's `status` and `q` filters, sorting, and `page`/`pageSize`
      pagination, returning `{ data, meta }`.
- [x] Write the seed script.
- [x] Map `P2002`, `P2025`, and `P2003` into your existing M05 error handler (not a new one).

## Verify

- [x] `migrate reset` plus the seed script rebuilds the database from scratch, every time.
- [x] The Bruno collection from M05 is still green against v2.
- [x] You can explain `migrate dev` vs `migrate deploy`, unaided.
- [x] `P2002`, `P2025`, and `P2003` each map to the right status code through your existing error
      handler.
- [x] `package.json` locks `prisma` and `@prisma/client` to `^7`, and every one-off command uses
      `npx prisma@7`.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": your schema-to-Prisma translation choices, what
      you had to hand-edit in the `--create-only` migration and why `migrate dev` couldn't generate
      it, and how you mapped Prisma error codes onto your M05 error handler.
- [x] "How to build it (teach it to the next trainee)": write the add-a-column-to-an-existing-table
      guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": migration conflicts, the hand-edited `--create-only` migration, the
      `better-sqlite3` install if it gave you trouble.
- [x] "Checkpoint evidence": `migrate reset` plus the seed script rebuilding the database from
      scratch, the Bruno collection from M05 still green against v2, `P2002`/`P2025`/`P2003` each
      mapping to the right status code, your explanation of `migrate dev` vs `migrate deploy`, and
      `package.json` locking `prisma`/`@prisma/client` to `^7`.
- [x] Close out "What I'd do differently".
