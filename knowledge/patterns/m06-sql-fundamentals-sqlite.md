# Patterns — discovered during M06 (2026-10-20)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Every artifact self-asserts session-scoped settings — silent-off is the worst failure mode

- SQLite's `PRAGMA foreign_keys` is per-connection and defaults OFF in every new session.
  schema.sql and seed.sql asserted it; exercises.sql didn't, so its `ON DELETE CASCADE`
  did nothing in a fresh CLI run: exit 0, no warning, junction rows orphaned — and the only
  thing that noticed was a `GRADED:` comment expecting 28 rows and reading 30.
- Generalize: any setting that doesn't persist (SQL PRAGMAs, session variables, dot-commands,
  env-dependent config) must be asserted **first line, in every artifact that depends on
  it** — not just in the "main" file. Enforcement failures are unique in that success and
  failure look *identical from the outside* unless you count rows.
- Contrast with errors that DO shout: CHECK/UNIQUE violations print and stop. A missing
  error is not evidence of a passing check — count the positive case too.

## Graded expectations written before running = an executable spec for teaching material

- Each of the 21 exercises carried a `GRADED:` comment (row counts, ids, first row) written
  before execution; running the artifact then diffed against its own comments. Two real
  defects fell out pre-PR: the silent-FK bug above, and Q20's expected "4 done tasks" where
  the true answer was 5 because Q16 flips a status earlier in the same ordered file —
  arithmetic no prose review would catch.
- The pattern's edge: the comment must be written from the *data*, not from memory of the
  query's intent (Q20's comment was written from intent — "the done tasks I seeded" — and
  missed the mutation chain). Derive expected values by composing the whole ordered run.
- For trainees: "write the expected result first, run, diff" turns every exercise into a
  three-way consistency check between query, data, and understanding — finding *which* of
  the three is wrong is the actual lesson.

## When a new layer encodes an old layer's rule, compare semantics, not names — at plan time

- Plan §12 said "keep the 409 rule, seed unique titles." I built *per-owner* uniqueness
  (`UNIQUE(user_id, title)`) with two cross-owner "Buy milk" seed rows — but M05's 409 is
  *global* (`hasTitle` over a userless store). Same rule name, different semantics; the
  drift surfaced only because I flagged it mid-build, and the fix cost a follow-up PR.
- Cheap version of the check: when a constraint/table/flag in layer N+1 restates a rule
  from layer N, write both rules as predicate sentences and diff them — "no two tasks share
  a title" vs "no two tasks *of one owner* share a title" — the words "of one owner" are
  the whole decision. Carry-overs named in a retro still need this; naming ≠ having read it.
- Resolution shape that worked (owner-approved): **keep the stricter old-layer rule where
  users meet it (API 409 global), the truthful new-layer rule in the DB (per-owner), and
  seed data that violates neither** — three deliberate layers, each provable, cross-owner
  behavior moved to a live proof (constraint-proofs TEST 6) instead of seeded rows the API
  would reject.

## Seed data is a teaching instrument — shape it for the queries you intend to teach

- Every "empty" in the seed exists to make a category of answer non-vacuous: a user with
  zero tasks (LEFT JOIN shows a real NULL row; `COUNT(*)` vs `COUNT(t.id)` diverges), a tag
  with zero uses (HAVING over a LEFT-joined 0), untagged tasks (32 rows vs 30 — INNER vs
  LEFT made countable), NULL and non-NULL due dates (three-valued logic bites in Q1, is
  redeemed in Q3), explicit ids and a spread of timestamps (OFFSET and ORDER BY answers are
  deterministic and checkable).
- Random filler data produces exercises with accidental answers: counts that happen to be
  equal, joins that happen to be full, HAVING clauses that filter nothing. Design the data
  *backwards* from the statements you want learners to be able to verify.
- Determinism is part of grading: stable ids + explicit sort keys = expected answers that
  hold on every rerun, which is what makes a `GRADED:` comment checkable rather than
  approximately right.

## Composite PK junction: one clause, one truth — surrogate needs three clauses for the same rule

- `task_tags (task_id, tag_id) PRIMARY KEY` states the pair is unique AND both columns
  non-NULL, in the key declaration. The surrogate-id alternative (`id` + `UNIQUE(task_id,
  tag_id)` + two `NOT NULL`s) encodes the same invariant across three clauses — more surface
  for drift, and the id column advertises a per-row identity the junction doesn't have.
- Junction-row reality test: "is there a row that means *this* AND *this*?" — a fact, not an
  entity. If you can't say what attributes a row would carry beyond the two references, it's
  a fact: composite PK, two columns, nothing else. Entities get surrogates; facts get keys.
- FK actions off the junction are policy per side: CASCADE toward the task (meaningless
  without it), RESTRICT toward the tag (don't silently rewrite in-use data) — chosen by
  asking "what should survive the delete," not copied from a tutorial.

## Parameter binding needs an API — the CLI can't demonstrate it, so the demo goes to node:sqlite

- The entire fix-vs-attack contrast is *two channels*: SQL text sent separately from values
  (`prepare(...).all(params)`). The sqlite3 CLI pastes strings either way (it has a
  `.parameter` mechanism, but its default idiom is exactly the vulnerability), so a CLI
  demo would show string-pasting twice and teach nothing about the mechanism.
- Node 24's built-in `node:sqlite` made the module zero-install: no better-sqlite3, no
  build tools, no package.json for a SQL-first module. The ExperimentalWarning on stderr is
  expected — document it in the material or beginners read it as "the attack failed."
- The three-round shape is the unit of proof: (1) vulnerable bypasses, (2) same payload
  parameterised is blocked, (3) legitimate input still works through (2). Round 3 is the
  one beginners omit and the one that kills the misreading "parameters block logins."

## Route around the environment instead of fighting it: choco admin-locked → official zip

- `choco install sqlite` died on a NuGet lock under `C:\ProgramData` (no admin on this
  harness). The official `sqlite-tools-win-x64` zip extracted to `~/.local/bin` (already on
  PATH) took a minute, needed no elevation, and produced the exact tool the brief names.
- Generalization for this program's "free tools only" constraint: prefer the vendor's
  portable artifact over the package manager — portable zips fail in visible ways (missing
  from PATH) while package managers fail in environmental ways (locks, elevation) that
  waste time looking like product bugs. Offer both routes in trainee material; lead with
  the one that cannot hit the permission wall.

## Validate diagrams by rendering them, not by reading them

- The ERD's `PK_FK` flags were the one artifact no grep could vouch for: Mermaid's docs
  list `PK`/`FK`/`UK`, and if `PK_FK` didn't parse, the README ERD would silently be a
  broken image on GitHub while every text check passed. It rendered fine (Mermaid 11
  supports multiple key flags per attribute) — but only a render *proved* it.
- The cheap harness that works on this Windows harness: a one-file HTML page with the
  diagram in `<pre class="mermaid">`, `await import()` of Mermaid's ESM from CDN,
  `mermaid.render()`, verdict stashed in `data-result` on `document.body`; then
  `chrome --headless=new --virtual-time-budget=30000 --dump-dom` and grep the attribute.
  No npm install, no puppeteer download — system Chrome was already there.
- Dead end worth remembering: `mermaid.ink` (the render-as-a-service URL) is
  Cloudflare-blocked from this harness (HTTP 1010). Local headless Chrome is the reliable
  path.
- Rule for future modules: any fenced diagram (Mermaid ERD, flowchart) that ships in
  learner-facing material gets rendered headlessly at /validate with its exact shipped
  text — "it looks right" is how broken fences ship.
