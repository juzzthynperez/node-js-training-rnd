# M01 Write-Up — Dev Environment, Git & GitHub

> Shared practical repository: https://github.com/juzzthynperez/node-js-training-ground
> Module directory: `modules/m01-dev-environment-git-github/`
> Filled in as the build progressed. Only what was actually run and observed is recorded here.

## What I built

A single shared **public** repository, `node-js-training-ground`, created once for this and
every future module, with this module's work isolated in `modules/m01-dev-environment-git-github/`.

Inside that directory:

- `hello-node.js` — a plain-JavaScript script that prints `Hello, Node.js!`
- `.gitignore` — covers `node_modules/`, `.env`, `*.db` (+ journal/wal/shm), the generated Prisma
  client (`prisma/generated/`, `node_modules/.prisma/`, `node_modules/@prisma/client`), `dist/`,
  `coverage/`
- `.env.example` — placeholder `NODE_ENV`/`PORT`, committed instead of a real `.env`
- `.nvmrc` — pins `24` (the cohort's Node 24 LTS)
- `README.md` — how to run the script

The change went through a real branch → PR → merge cycle:

**Merged PR:** https://github.com/juzzthynperez/node-js-training-ground/pull/1
(state `MERGED`, merged 2026-10-13 via merge commit `4c7f851`)

Branch history as merged (verified with `git log --oneline --graph`):

```
*   4c7f851 Merge pull request #1 from juzzthynperez/feat/m01-hello-node
|\
| *   45cd517 chore: resolve greeting-line conflict   ← conflict resolved here
| |\
| | * 9cff5c3 feat: use casual greeting               ← conflict/m01-alt-greeting
| * | fd93f58 fix: use full product name in greeting  ← feat/m01-hello-node
| |/
| * 10e2cd9 refactor: extract greeting message into a function
| * 8c5e812 docs: add run instructions to module readme
| * 4f30465 feat: add hello-node script
|/
* 8d7c957 chore: scaffold m01 module directory
```

Five Conventional Commits on the PR branch (feat, docs, refactor, fix, feat) plus the conflict
resolution merge commit — the DoD asks for 3 or more.

## Why it's built this way (key decisions)

**One shared repository, not one per module.** The module directory acts as the boundary:
because a nested `.gitignore` applies only to its own directory, each future module can carry
its own ignore rules without touching the others. A separate repository per module would mean
repeated repo setup, N remotes to maintain, and no place for cross-module work (the capstone's
8+ PRs, peer review) — so the default is reuse, and a new repo is only worth proposing if a
module needs something a directory cannot give it, such as its own CI secrets or publishing
permissions. M01 needs none of that.

**Merge commit, not squash.** `gh pr merge 1 --merge` was used deliberately. Squashing would
have collapsed the branch into one commit and erased the conflict-resolution commit from
history — but "a resolved merge conflict in the PR's history" is the deliverable itself.

**The conflict was between two of my own branches.** From `feat/m01-hello-node` I branched
`conflict/m01-alt-greeting` and changed the greeting line to `'Hi from Node!'`; back on the
feature branch I changed the same line to `'Hello, Node.js!'`. Git could not auto-merge one
line two ways, so the merge stopped with `CONFLICT (content)`.

**How I resolved it:** opened the file, found the markers, deleted `<<<<<<<`, `=======`,
`>>>>>>>` and kept `return 'Hello, Node.js!';` — the fuller, correct name — then staged the file
and committed the merge as `chore: resolve greeting-line conflict`.

**Merge vs rebase — would it have changed anything?** Merging preserved the true shape: both
branches and the exact point where they collided remain visible in the graph. Rebasing
`conflict/m01-alt-greeting` onto the feature branch would have replayed its single commit on
top, forced the same conflict during `rebase` (resolved with `git rebase --continue` instead of
`git merge --continue`), and rewritten that commit into a new one — the graph would look linear
and hide that two lines of history had diverged. For a learning deliverable whose evidence *is*
the conflict, merging showed more.

**What would differ on a private repo with a no-direct-push rule.** Nothing about the branch →
PR → merge flow itself — that is GitHub Flow either way. What changes: branch protection (require
PRs, required reviews, dismiss stale approvals) would enforce it instead of convention; on a
free plan that enforcement on *private* repos is plan-dependent, which is exactly why the program
standardises on public repos. With a second trainee, the PR would also carry a required
reviewer — that part is missing here (see "What tripped me up").

## How to build it (teach it to the next trainee)

A guide to taking one change through branch → PR → merge, including a real conflict. Every
command below was run in this module.

**0. One-time setup for the shared repo** (skip if it exists):

```bash
gh auth status                       # confirm you're logged in
mkdir node-js-training-ground && cd node-js-training-ground
git init -b main
gh repo create <you>/node-js-training-ground --public --source . --push
```

**1. Scaffold on `main`, then branch before writing code:**

```bash
git checkout -b feat/m01-hello-node
git add modules/m01-dev-environment-git-github/hello-node.js
git commit -m "feat: add hello-node script"        # Conventional Commits, every time
```

*Reasoning:* `main` stays releasable; the PR becomes a reviewable unit of work. Commit the
`feat` while the idea is fresh; run `git status` before each `git add` so nothing unintended
rides along.

**2. Keep committing on the branch** (this PR had `docs:`, `refactor:`, `fix:` — five
Conventional Commits total; the DoD needs three).

**3. Engineer the conflict.** Branch off your feature branch, change a line, go back, change the
*same line differently*, then merge:

```bash
git checkout -b conflict/m01-alt-greeting
#   edit: return 'Hi from Node!';      then:
git add modules/m01-dev-environment-git-github/hello-node.js
git commit -m "feat: use casual greeting"

git checkout feat/m01-hello-node
#   edit the same line: return 'Hello, Node.js!';   then:
git commit -am "fix: use full product name in greeting"

git merge conflict/m01-alt-greeting
#   -> CONFLICT (content): Merge conflict in .../hello-node.js
```

**4. Resolve by hand.** Open the file; you'll see:

```
<<<<<<< HEAD
  return 'Hello, Node.js!';
=======
  return 'Hi from Node!';
>>>>>>> conflict/m01-alt-greeting
```

Keep the version you want, delete all three marker lines, then:

```bash
git add modules/m01-dev-environment-git-github/hello-node.js
git commit --no-edit -m "chore: resolve greeting-line conflict"
git log --oneline --graph          # both branches, one merge node
```

*Reasoning:* markers are Git asking "which side wins?" — the answer is yours to make, and you
can also keep parts of both. `git merge --abort` is the escape hatch if you get lost.

**5. Push and open the PR:**

```bash
git push -u origin feat/m01-hello-node
gh pr create --title "M01: hello-node script with resolved merge conflict" --body "..."
```

**6. Merge with a merge commit — never squash here:**

```bash
gh pr merge 1 --merge --delete-branch
gh pr view 1 --json state          # -> "MERGED"
```

**7. Verify on `main`:**

```bash
git checkout main && git pull
git log --oneline --graph                    # history intact, conflict commit visible
fnm exec --using 24 -- node modules/m01-dev-environment-git-github/hello-node.js
#   -> Hello, Node.js!
git log --all -- "**/.env"                   # empty: no secret ever committed
```

## Concepts worth explaining

**The working tree → staging area → commit model.** Three stores, in order. The *working tree*
is the files on disk — `git add` copies a snapshot of chosen changes into the *staging area*
(the index), your concrete proposal for the next commit. `git commit` seals the staged snapshot
into the repository permanently. This is why `git status` has two sections (staged / unstaged):
you can edit freely, stage only what belongs together, review with `git diff` (working vs
staged) and `git diff --staged`, and rewind files you regret touching with `git restore`.
Commits are snapshots, not diffs — each points at the whole tree plus a parent, which is what
makes `git log` a walkable history graph.

**`git revert` vs `git reset`.** Both undo, but they differ in what they touch.
`git revert` makes a **new commit** that applies the inverse of an older one — history only ever
grows, so it is safe on shared branches: everyone sees "commit X, then commit -X".
`git reset` **moves the branch pointer** — `--soft` keeps the undone changes staged, `--mixed`
(default) leaves them in the working tree, `--hard` discards them entirely — so commits
disappear from the branch. Rewriting history anyone else has based work on is the danger:
`reset` is for local cleanup (that bad commit never pushed), `revert` for anything public.
Rule of thumb: shared → revert, private → reset, and `--hard` throws work away, so check
`git status` first.

## What tripped me up

Recorded only what actually happened in this build:

- **Local Node was v22.23.2, the cohort pin is 24.** The default fnm version differed from
  `.nvmrc`, so a plain `node ...` would have run the wrong version. `fnm use` only affects a
  shell that has fnm's env loaded, so for a one-shot verified run I used
  `fnm exec --using 24 -- node ...` — observed output: `v24.21.0`.
- **The merge strategy must be named explicitly.** I passed `--merge` on purpose: had this PR
  been squashed instead, the conflict-resolution commit would have vanished from `main`.
  `gh pr merge` prompts for a strategy (merge commit, squash, or rebase), so the safe habit is
  to always name it on the command line rather than rely on a prompt.
- **CRLF warnings on every `git add`** ("LF will be replaced by CRLF") on Windows. Cosmetic
  here (`core.autocrlf`), but it makes output noisier than expected.
- **Auto-merge sounded scarier than it was.** Git reported `CONFLICT (content)` and left
  `UU` in `git status`; the actual fix was deleting three marker lines.
- **No peer reviewer.** The architecture doc assumes at least two trainees; there is none in
  this environment, so the PR was opened and merged by its author. The conflict was created
  between two of my own branches (which the brief explicitly allows) — a planted conflict from
  a partner would have exercised the review side too. Flagged as a known deviation.

## Checkpoint evidence

- **Merged PR:** https://github.com/juzzthynperez/node-js-training-ground/pull/1 —
  `gh pr view 1 --json state` returned `MERGED` (2026-10-13, merge commit `4c7f851`);
  branch carried 5 Conventional Commits + 1 conflict-resolution merge commit (DoD: 3+). ✔
- **`.gitignore` present** at `modules/m01-dev-environment-git-github/.gitignore`, covering
  `node_modules`, `.env`, `*.db`, the generated Prisma client, `dist`, `coverage` — verified
  tracked in `main` via `git ls-tree -r main`. ✔
- **`.env.example` present** at `modules/m01-dev-environment-git-github/.env.example`, and
  `git log --all -- "**/.env"` returned nothing — no real `.env` in history. ✔
- **`git revert` vs `git reset`** — explained in my own words above (revert = new undoing
  commit, safe for shared history; reset = moves the branch pointer, `--soft/--mixed/--hard`
  control what happens to the changes, history rewrite, local only). ✔

## What I'd do differently

- Record the run of `fnm use` inside a properly initialised shell once, so the commands in the
  write-up are exactly what a trainee types rather than the `fnm exec --using 24` workaround.
- Seed `.env.example` with the keys a real consumer expects sooner — it was written as an
  empty habit, and M03+ will want `PORT` semantics pinned down earlier.
- Have the conflict partner branch created *before* the second edit, and show the
  `git diff` of both sides in the write-up — the screenshots of *why* the two edits collided
  teach faster than the resolution itself.
- If a peer trainee is available next cohort, hand them the PR for review rather than merging
  it solo, so the module's peer-review half is genuinely exercised.
