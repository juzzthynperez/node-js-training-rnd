# M01 Write-Up — Dev Environment, Git & GitHub

**Trainer:** juzzthynperez · **Date:** 2026-10-16 · **Repo:** [juzzthynperez/node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground) (public) · **Merged PR:** [#1 — feat: add hello-node program with resolved greeting conflict](https://github.com/juzzthynperez/node-js-training-ground/pull/1)

> Written as the work happened, from a live evidence log kept outside the repo
> (`evidence/m01/`, gitignored from the first commit so nothing private could be pushed by
> accident). Where a claim matters, it links to the public artifact that proves it.

## What I built

- **Public training repo** `node-js-training-ground`, created empty and pushed with the
  ignore rule (`evidence/`) already in the root commit — the defensive move came *before*
  there was anything to defend against, which is the whole point of the habit.
- **Toolchain verified end to end** (not just installed): VS Code 1.142.0 (`code
  --version`), Git 2.55.0, Node v22.23.2 managed by **fnm 1.39.0** (which also has the
  cohort's v24.21.0 installed and ready), and **Bruno 4.2.1** for API testing. Two real
  findings from verifying instead of assuming: fnm — not nvm — is the version manager on
  this Windows machine, and Bruno is a GUI app with **no `bru` CLI on PATH**, so it can't be
  version-checked from the terminal at all; I confirmed it via winget/installed-apps
  instead. A checklist item is only done when you've actually made the tool answer.
- **`.nvmrc` pinning Node 24** (the cohort version) and **`.env.example`** — committed on the
  feature branch in one `chore:` commit.
- **`hello-node.js`** — prints a greeting and the runtime version. Ran it before committing:
  `node hello-node.js` → `Hello from Node.js trainee! Running on Node v22.23.2`.
- **A real merge conflict**, planted deliberately: two branches (`feat/m01-conflict-a`,
  `feat/m01-conflict-b`) rewrote the same greeting line from the same parent — "Welcome
  from" vs "Hi there from". `git merge` refused with `CONFLICT (content)`, status `UU`,
  markers on lines 3–7. Resolved by combining both wordings; the script ran clean
  before I committed the merge.
- **Merged PR #1** into `main` carrying 5 Conventional Commits, with the conflict and its
  resolution visible in the PR history.

## Why it's built this way (key decisions)

- **One shared public repo for the whole program** (`training-ground`), rather than a fresh
  repo per module. The supervisor brief describes "your own public training repo"; the
  program's two-repo convention refines that — code and experiments live in the shared
  ground, documentation lives in `node-js-training-rnd`. One remote to remember, one PR trail
  to audit, and every module's practice accumulates in one place.
- **Branch structure:** everything on `feat/m01-conflict-a` (plus the two throwaway conflict
  branches) — nothing touched `main` directly. The conflict branches were cheap: create from
  the same parent, change one line each, merge, delete later.
- **Private-vs-public repo difference:** with a private repo and a no-direct-push rule,
  branch protection is plan-gated (the reason the program standardizes on public repos).
  On public, I relied on self-discipline for the same workflow — the only real difference is
  who enforces the rule: GitHub's settings, or me.
- **Conflict resolution:** kept both wordings combined rather than picking a winner — the
  resolution only has to be *correct* and *runnable*; the lesson is the process, not the
  greeting. Merge (not rebase) kept both branch sides visible in history, which is exactly
  what a reviewer needs to see that a conflict existed.
- **Ignore-first ordering:** `evidence/` went into `.gitignore` in the root commit, before a
  single note existed. Rules added *after* the near-miss are rules that already nearly failed.

## How to build it (teach it to the next trainee)

A change travels this path: **branch → stage → commit → PR → review → merge.**

1. `git checkout -b feat/<thing>` — leave `main` alone until a PR exists.
2. Do a *small* piece of work. Run it. Really run it — `node hello-node.js` — before staging.
3. `git add <file>` then `git commit -m "feat: …"` — Conventional Commits from commit one,
   not retrofitted.
4. Repeat 2–3 until the piece is complete (this PR had 5 commits; three is the floor).
5. `git push -u origin <branch>`, open the PR, describe *what and why*.
6. Merge. `main` now has your change; delete or reuse the branch.

**The conflict, reasoning included:** both branches started at the same commit and edited the
same line — that is the recipe. When the merge stopped with `CONFLICT (content)`, the file
showed `<<<<<<< HEAD` … `=======` … `>>>>>>> other-branch`: your side, their side. Fix the
file so it's correct code (not "a side"), run it, `git add`, and commit — the commit message
says *how* you resolved it, because that's what a reviewer can't guess. Then merge.
An interactive rebase could have replayed one side onto the other, but it rewrites history;
merging preserves the conflict as evidence — better for learning, required for shared branches.

## Concepts worth explaining

**The working tree → staging → commit model.** Git has three rooms. The *working tree* is the
files you edit. The *staging area* (`git add`) is a docket of exactly what will go into the
next commit — it lets you shape a commit instead of dumping whatever changed. A *commit* is a
frozen snapshot plus a message, linked to its parent. `git status` is simply "which room is
each file in?"; `git diff` compares rooms. Most beginner confusion is asking "is this change
saved yet?" when the real question is "in which room is it?"

**`git revert` vs `git reset`.** `revert` makes a *new commit* that undoes an old one —
history grows, nothing is rewritten, safe on shared branches. `reset` *moves the branch
pointer backward* — the undone commits vanish from the branch (recoverable only while the
reflog remembers them), fine on a private branch, dangerous once others have pulled. Rule of
thumb: shared branch → `revert`; your own unpushed mistake → `reset`.

## What tripped me up

- **The `.nvmrc` says 24 but `node --version` says v22.23.2.** Pinning the file doesn't switch
  the runtime — a version manager does. Easy to write `.nvmrc`, feel done, and never notice
  your shell is still on another version. `hello-node.js` printing `process.version` caught it
  immediately — print your runtime version, always.
- **CRLF warnings on Windows** (`warning: LF will be replaced by CRLF`) look alarming but are
  line-ending normalization, not a problem. First time, I assumed something was broken.
- **Conflict markers silently break the script.** After the conflict, the file *contains*
  `<<<<<<<` lines — running it fails confusingly if you try before cleaning the file. The fix
  isn't only choosing words; it's deleting every marker line.
- **`git restore` does different things depending on the file's room.** While practicing
  restore/stash for real, I stashed a *staged new* file and, after `git stash pop`, ran
  `git restore` on it expecting it to vanish — it stayed. That's correct: `restore` reads
  from the *index*, and the index still held the staged copy. On a *tracked, edited* file it
  works as everyone expects — I verified it properly this time by hash: after restoring an
  edit to `hello-node.js`, the working blob matched `HEAD:hello-node.js` exactly
  (`b2ed0f61` on both sides). "Undo" in Git always means "restore to some room's version" —
  the flag (or lack of it) decides *which* room.
- **`git merge` prints `Automatic merge failed` even when the conflict is trivial** — the
  emotional weight of the message is not proportional to the actual work.

## Checkpoint evidence

- **Merged PR with 5 Conventional Commits (≥3 ✓):**
  https://github.com/juzzthynperez/node-js-training-ground/pull/1 —
  history shows both conflict sides (`bec016e`, `d41af64`) and the resolution (`e5a5d1f`).
- **`.gitignore` covering the Node list ✓** —
  [.gitignore](https://github.com/juzzthynperez/node-js-training-ground/blob/main/.gitignore)
  covers `node_modules`, `.env`, `*.db`, Prisma generated client (`src/generated/`), `dist`,
  `coverage` (plus the program's private `evidence/`).
- **`.env.example` present ✓** —
  [.env.example](https://github.com/juzzthynperez/node-js-training-ground/blob/main/modules/m01-dev-environment-git-github/.env.example),
  with a comment explaining the habit even with nothing to configure yet.
- **`git revert` vs `git reset`** — narrated above, unaided.
- Private evidence (command logs, conflict markers as-planted, toolchain versions) lives in
  `evidence/m01/` in my local clone — deliberately never committed.

## What I'd do differently

- **Switch the runtime first.** Verify the version manager actually selects the `.nvmrc`
  version *before* building anything, instead of noticing through the program's output.
- **Commit the README with real content up front** — I seeded one line; a short "what goes in
  this repo / what doesn't" saves the next trainer (or me, next module) a guess.
- **Clean up the conflict branches immediately after the merge** — they're evidence for the
  write-up, so I kept them; next time, capture the evidence, write it up, then delete the
  branches the same day.
- **One commit per concern from the start** — my first `chore` bundled `.nvmrc` and
  `.env.example`. Related, but a reviewer interested in the Node pin had to take the
  `.env.example` change along for the ride.
