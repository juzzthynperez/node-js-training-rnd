# Node.js Developer Training

This repo holds the Build-to-Teach version of the Node.js Developer Training Program: nine training
stages in `modules/` (M01–M09), followed by a capstone (M10). The format is different on purpose:
instead of being taught the material, the trainee builds toward each stage's deliverable from a
brief, and writes up how they did it as they go. The write-up is what ends up teaching the next
cohort.

A note on words: a **module** in this repository is a training stage, not a JavaScript/Node.js
module. Stages M02 and M03 are the ones that actually teach what a JavaScript/Node.js module is
(named vs default exports, ESM vs CommonJS).

## What is in this repository

| Path | What it holds |
| --- | --- |
| `modules/` | The nine stage folders (M01–M09), each with a brief, a checklist, and a write-up template |
| `modules/m10-capstone/` | The capstone, with its own brief, checklist, and write-up template |

This repository is documentation only. All hands-on code for every module lives in the
companion public repository [node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground),
where module exercises are committed and their branch/PR history is the checkpoint trail.
Trainer-private evidence captures stay local to that repo's gitignored `evidence/` folder and
are never committed.

## Ground rules (every stage)

- **Free tools only. No Docker, no paid licences.**
- **SQLite is the only database you touch hands-on.** PostgreSQL and MongoDB are covered as concepts
  in M07, with no labs.
- **Plain JavaScript in Week 1. TypeScript from M04 onward.**
- **Pinned versions for the whole cohort:** Node.js 24 LTS, Express 5.x, TypeScript 7.0,
  Prisma ORM 7.x, Zod 4.x, Jest 30.x, Supertest 7.x. Your trainer re-verifies these are still current
  immediately before the cohort starts (versions move fast) but the choice of Node 24 over 26 is
  already decided — see `m04-typescript-on-node/brief.md`. Prisma is always installed and invoked as
  `@7` — an unversioned `prisma`/`npx prisma` install may resolve to Prisma 8, which does not read
  `schema.prisma` for this stack.
- **Public training repos** for every stage — see `m01-dev-environment-git-github/brief.md`.
- **Bruno** for manual API testing. Collections are plain text, committed to `bruno/` in your repo,
  never left uncommitted.
- **Git/GitHub workflow throughout:** feature branches, pull requests, and Conventional Commits.

## How a stage works

1. **Your trainer hands you the brief.** Each stage's `brief.md` says what to build and how you
   will know it is done: objective, scope, stack constraints, the named deliverable, a lab, a
   definition of done, and what to confirm with your trainer before you start. It contains no
   lesson, no steps, no code, and no solutions. How to build it is yours to work out.
2. **You build and write up the stage together.** Fill in `write-up-template.md` in the stage
   folder while you build, not after the build works. The lab's "you build and capture" section is
   your deliverable; your write-up is the teaching content — write it for someone who will follow
   it with no trainer to ask.
3. **Your trainer reviews both together.** When you reach the definition of done, bring both the
   working system and the write-up to your checkpoint. The question is not only "does it work?" but
   "could someone else learn from this?"
4. **You revise.** Fix what the review finds, in both the build and the write-up, before starting
   the next stage.
5. **The finished pair goes into the content library.** An accepted stage is the build plus its
   write-up. Your trainer decides what carries into the next cohort's plan.

Review happens at the end of every stage, not just at the end of the course.

## Why review as you go, not at the end

A write-up finished after the deliverable already works tends to be thin — there's no real
pressure to make it good once the thing runs. Reviewing both pieces together, stage by stage,
means the trainer is judging whether the work is teachable, not just whether it works, and catches
problems early instead of in a finished draft nobody wants to redo.

## What is in each stage folder

| File | Who writes it | Purpose |
| --- | --- | --- |
| `brief.md` | Your trainer | Objective, scope, stack constraints, the deliverable, a lab (goal, steps, what to capture), and definition of done. No answers. Do not edit it. |
| `write-up-template.md` | Your trainer | The blank seven-section structure. Stays blank as the reusable template for the next cohort. |
| `write-up.md` | You | Your filled-in write-up, drafted as you build (never after), following the template's sections — your real experience, decisions, and checkpoint evidence. |
| `training-material.md` | You | The learner-facing module designed from your experience: explanations, examples, exercises, troubleshooting, and knowledge checks for the next cohort's trainees. |
| `tasks.md` | Generated from `brief.md` via `/trainee-task-planner` | An ordered checklist mirroring the brief — setup, then research (M02–M09), then build (Day 3/4/5 for the M10 capstone), then verify, then write-up. You tick the boxes; it holds no answers and no steps you weren't already told. |

Your Node.js code lives outside this repository, in the public
[node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground) repository
you set up in M01. Link it from the header of your write-up. The capstone folder follows the same
convention.

## Stages

| Stage | Title | Theme | Leaves behind |
| --- | --- | --- | --- |
| [M01](modules/m01-dev-environment-git-github/brief.md) | Dev Environment, Git & GitHub | Toolchain, the everyday Git workflow, a merged PR with a resolved conflict | Nothing |
| [M02](modules/m02-modern-javascript-async-await/brief.md) | Modern JavaScript & Async/Await | Language essentials, the event loop, reliable async code | The Async toolkit (ported to TypeScript in M04) |
| [M03](modules/m03-nodejs-runtime-npm/brief.md) | Node.js Runtime & npm | Core modules, npm, a bare `node:http` server | Nothing |
| [M04](modules/m04-typescript-on-node/brief.md) | TypeScript Basics & TypeScript on Node.js | Strict TypeScript, the Task domain types | The ported toolkit and Task domain types |
| [M05](modules/m05-express-task-api-v1/brief.md) | Express: Routing, Middleware & Error Handling | REST API structure, middleware, one consistent error model | Task API v1 (extended through M07–M09) |
| [M06](modules/m06-sql-fundamentals-sqlite/brief.md) | SQL Fundamentals with SQLite | Relational design, joins, transactions, SQL injection | The schema design (translated into Prisma in M07) |
| [M07](modules/m07-prisma-orm-migrations/brief.md) | Prisma ORM & Migrations | Migrations, the typed client, Prisma error mapping | Task API v2 |
| [M08](modules/m08-zod-validation/brief.md) | Zod Validation & Type Inference | Runtime validation, type inference from schemas | Task API v3 |
| [M09](modules/m09-integration-testing/brief.md) | Integration Testing with Jest + Supertest | Real-database integration tests, CI | Task API v4 (handed off to the M10 capstone) |
| [M10](modules/m10-capstone/brief.md) | Capstone Project | Independent API, authentication, a provable concurrency fix | — (terminal stage) |

Work the stages in order. Later stages build on earlier ones: M05's Task API carries forward
through M07, M08, and M09 (v1 → v4), and M06's schema design is what M07 translates into Prisma.
TypeScript (M04) comes before Express (M05) so the API is typed from the start; SQL (M06) comes
before Prisma (M07) so migrations aren't magic; hand-written validation (M05) comes before Zod
(M08) so the payoff is felt. Folder names are descriptive; each brief refers to other stages by
module ID (M01–M10).

## Assessment

- Every module ends in a checkpoint (see that stage's `brief.md`).
- The capstone (M10) is scored out of 100 against a fixed rubric; 70 or more passes. The rubric
  lives in `modules/m10-capstone/brief.md` — it is the one place point values are specified, and it
  is not a solution, it is how your finished API gets graded.
- By the end of the program you should be able to evidence all ten competencies listed in the PRD's
  Section 6 — each module's `brief.md` names which one(s) it builds toward.
