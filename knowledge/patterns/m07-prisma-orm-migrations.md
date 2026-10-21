# Patterns — discovered during M07 (2026-10-21)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## A frozen test suite proves what it asserts — never the contract around it; diff old behavior, don't just re-run it

- The M05 Bruno suite passed 15/15 against v2, and the build was still two silent contract
  drifts deep: (1) M05's update pre-checked duplicate titles *before* existence, so
  `PATCH <missing-uuid> {"title":"Buy milk"}` answered **409** — v2's naive Prisma `update`
  answers **404** (P2025 fires first); (2) v1 echoed an explicitly-supplied `""` description
  back, v2 renders `""` as absent (the M06 column is `NOT NULL DEFAULT ''` and cannot tell
  "cleared" from "never set"). Neither case is in any `.bru` file; both were found by *reading
  the old implementation* line by line during the pre-PR review.
- Generalize: a regression suite is a sampling of the contract, and the unasserted remainder —
  error *precedence*, empty-string rendering, header echo, ordering — is exactly where a
  layer-swap drifts silently. The review that matters diffs BEHAVIOR of layer N against layer
  N+1 (old source vs new source, case by case), with the suite as a floor, not the check.
- The two fixes split cleanly and worth memorizing as a pair: drift (1) was **reversible** —
  restore the old ordering with an explicit pre-check, cost one extra query per PATCH; drift
  (2) was **irreducible** at this layer (no wire-level signal exists in the data model), so the
  honest fix is documenting the difference, not faking it. "Restore or document" beats
  "silently inherit".

## A schema diff carries structure, never values — the hand-edit is where data knowledge lives

- Adding required `priority` to a populated `tasks` table: `migrate dev` refused outright
  ("Added the required column … 4 rows … not possible to execute this step") and the
  `--create-only` scaffold's `INSERT … SELECT` copied every column *except* `priority` — it
  knew the shape of the destination and nothing about the journey. The shipped migration
  differs from the scaffold in exactly one construct: a `CASE` expression deriving each row's
  priority from its `due_date` (overdue→high, NULL→low, else→medium), with a fixed timestamp
  literal because migrations replay forever and "now" doesn't.
- Generalize: three cases, decided before touching SQL — constant default acceptable (one
  `ADD COLUMN … NOT NULL DEFAULT`, SQLite-safe on data), derived value (backfill phase),
  derived + constrained + no default (full table rebuild, the only SQLite route). The tool can
  generate the *frame* for all three; the *predicate* is knowledge only you have.
- Edge that mattered twice: the rebuild's `DROP TABLE` performs an implicit `DELETE FROM`
  under enabled FKs and would have CASCADE'd `task_tags` out silently (M06's silent-FK class,
  one layer up). SQL cannot assert its own preconditions, so the assumption (junction empty at
  this point in history — verified, and true on every fresh replay) is *stated in the SQL
  header*. A migration is a historical document; its comments are its assertions.

## Schema and migration history are one unit — move either alone and the tool demands to invent the other

- To replay "populate between migration 1 and 2", I stashed migration 2 while `schema.prisma`
  still declared `priority`. `migrate dev` then saw schema-vs-history drift and stopped at an
  *interactive* "Enter a name for the new migration" prompt — which hung a non-interactive
  shell until timeout. The fix is not a flag; it's choreography: revert the schema field AND
  remove the migration folder together, do the era's work, restore both together.
- Adjacent trap from the same session, same root cause (history is a sequence the tool owns):
  `--create-only` wrote folder `…025324_add_priority` and I *guessed* a timestamp for my
  hand-edited copy (`…025428`), producing two same-named migrations. `migrate dev` applied the
  naive one first, hit the predicted NOT NULL failure, and left a poisoned history row
  (`finished_at NULL`) that blocks the next attempt until resolved. The tool creates folder
  names; a human-created sibling is a fork of history nobody asked for. Unapplied and wrong →
  delete the folder and replay (dev-only, cheap); applied and wrong → write a *new* migration
  that fixes it, never edit the recorded one.

## Pin direct deps to the dependency's own range — two copies of a native module is two builds, one of them unpracticed

- `npm install better-sqlite3` (unpinned) grabbed v13; `@prisma/adapter-better-sqlite3@7.10.1`
  requires `^12.6.0`. npm cheerfully hoisted BOTH — v13 at top level, v12 nested under the
  adapter — so the module carried two native bindings to compile/download, and the nested one
  was the one that fell through prebuild-install to a node-gyp failure. Reading the adapter's
  own `package.json` dependencies and matching the direct pin (`^12.6.0`) deduped everything to
  a single 12.11.1, verified with `npm ls`.
- Generalize: when a direct dependency is a *peer of something you import*, its version range
  is not a free choice — read the dependent's range and land inside it, or you silently sign
  up for duplicate native artifacts on every install. Related: npm's install-script gate
  blocked `prisma`, `better-sqlite3`, and `@prisma/engines` scripts with a warning, not a
  failure — "installed" ≠ "usable" until `npm install-scripts approve` + `npm rebuild` prove
  the binaries load (the require-test is the proof, not the exit code).

## Remembered tool behavior is a hypothesis — settle version-drift questions against the installed binary, including the reviewer's

- Both the builder (me) and the reviewer carried the same stale memory: "Prisma seeds
  automatically on `migrate reset`". Empirically on 7.10.1: reset finished, tables were empty;
  `prisma db seed` ran the configured `migrations.seed` fine. The DoD's "reset *plus* seed"
  became a chain (`prisma migrate reset --force && tsx prisma/seed.ts`) — load-bearing, not
  belt-and-braces. The reviewer's *live* re-verification of every other claim (fresh-clone
  replay, full Bruno, both proof scripts) is what makes the rest of the review trustworthy.
- Two reviewer findings were settled the same way instead of by argument: "`dev.db` isn't
  gitignored" → `git check-ignore -v` showed the ROOT `.gitignore` already covers `*.db`
  (false positive); "reset double-seeds" → the chain was tested and seeds *once* (inverted).
- Generalize: config-shape and version-behavior questions get answered from the installed
  artifacts (`@prisma/config/dist/index.d.ts` for where `seed` belongs; the CLI's own output
  for what runs it) — docs describe a version, the binary is the version. Reviewer claims get
  the same evidence discipline as builder claims; "I read it in the source" and "I ran it" are
  different evidence tiers, and the fixes chosen should match the tier.

## Deterministic seed identity is a contract between layers — constants in code require constants in data

- The service pins every API write to `DEFAULT_OWNER_ID`, and `seed.ts` *imports that same
  constant* to create its first user — the three-layer uniqueness rule (API global-409 ⊂ DB
  per-owner unique ⊂ globally-unique seed) only lines up because seed users are fixed
  v4-shape UUIDs, not `randomUUID()`. With random ids the constant cannot exist, the pinned
  FK trips P2003 on the first create, and the whole single-owner argument collapses to a
  runtime lookup nobody wrote.
- Generalize: deterministic seed data is usually justified as "reproducible evidence" (M06's
  lesson — stable ids make `GRADED:` comments checkable). M07 adds the stronger reason: **fixed
  data enables fixed references in code**, which is what lets a *policy* (who owns API writes)
  be a reviewable constant instead of a query. The seam to watch: two files that must agree on
  a magic value should share one source (the import), and a comment should name the other file
  in both directions.
