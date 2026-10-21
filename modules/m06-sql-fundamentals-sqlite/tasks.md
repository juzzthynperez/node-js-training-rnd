# M06 Tasks — SQL Fundamentals with SQLite

Design a small relational schema and write the SQL needed to query it, before an ORM hides it.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; have your M05 write-up on hand (you're replacing its
      in-memory store with this schema, then with Prisma in M07).
- [x] Install the `sqlite3` CLI or DB Browser for SQLite.
- [x] Continue the branch → PR → merge habit for this module's checkpoint.

## Research

- [x] The relational model: tables, rows, primary and foreign keys.
- [x] SQLite specifics: single-file database, dynamic typing and type affinity, `PRAGMA
      foreign_keys`.
- [x] DDL: `CREATE`/`ALTER`/`DROP` and the constraints `PRIMARY KEY`, `NOT NULL`, `UNIQUE`, `CHECK`,
      `DEFAULT`, `FOREIGN KEY … ON DELETE`.
- [x] DML: `INSERT`/`SELECT`/`UPDATE`/`DELETE`, filtering, sorting, `LIMIT`/`OFFSET`, and the idea
      of keyset pagination.
- [x] Aggregates with `GROUP BY`/`HAVING`, `INNER` and `LEFT` joins, and subqueries.
- [x] Modeling relationships: 1:1, 1:N, M:N with a junction table, and normalisation basics
      (1NF-3NF).
- [x] Indexes, `EXPLAIN QUERY PLAN`, and transactions/ACID (`BEGIN`/`COMMIT`/`ROLLBACK`).
- [x] SQL injection and parameterised queries.
- [x] How to document an ERD with Mermaid in a GitHub README.

## Build

- [x] Design the Task API schema: `users`, `tasks`, `tags`, `task_tags` — including the
      junction-table design for `task_tags`.
- [x] Write the DDL to create the schema, with appropriate constraints and foreign keys.
- [x] Seed the schema with SQL.
- [x] Write and run at least 20 of your own graded query exercises against your schema, covering
      filtering/sorting/pagination, `INNER`/`LEFT` joins, aggregates with `GROUP BY`/`HAVING`, and
      at least one subquery.
- [x] Write at least one query inside an explicit transaction (`BEGIN`/`COMMIT`/`ROLLBACK`) — no
      exercise strictly forces this, but you'll need the skill for real in the M10 capstone.
- [x] Write a vulnerable, string-concatenated query and demonstrate a successful SQL injection
      against it.
- [x] Fix that same query with parameters and demonstrate the injection now fails.
- [x] Draw the ERD in Mermaid in your README, matching the schema you actually built.

## Verify

- [x] 20+ exercises complete, covering every required category.
- [x] You can explain primary vs foreign keys, what an index is for, and how to model 1:N vs M:N,
      unaided.
- [x] The injection demo actually succeeds against the vulnerable query, and the parameterised fix
      actually blocks it.
- [x] The Mermaid ERD matches the schema you actually built.
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": your junction-table design for `task_tags`, what
      you indexed and why, and which of your 20+ exercises taught you something you didn't expect.
- [x] "How to build it (teach it to the next trainee)": write the junction-table guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": the SQL injection demo, any constraint or join surprises.
- [x] "Checkpoint evidence": your 20+ query exercises, the Mermaid ERD matching your schema, and
      the injection demo succeeding against the vulnerable query then failing against the fixed
      one.
- [x] Close out "What I'd do differently".
