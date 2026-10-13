# Training Material — M01: Dev Environment, Git & GitHub

> A beginner-friendly, standalone guide for **Week 1 · Day 1 · 4 hours**.
> Built from this module's `brief.md`, `tasks.md`, and the completed write-up. Every command
> here was executed and verified in the shared repository
> [`juzzthynperez/node-js-training-ground`](https://github.com/juzzthynperez/node-js-training-ground).
> Personal reflections live in the write-up, not here.

---

## Learning objectives

By the end of this module you will be able to:

1. Explain the three Git stores — working tree, staging area, commit — and say what each
   command (`add`, `commit`, `status`, `diff`, `restore`, `stash`) moves between them.
2. Create a repository, branch off `main`, and take a change through **branch → pull request →
   merge** on GitHub.
3. Deliberately create a merge conflict, resolve it by hand, and explain what Git was asking.
4. Write **Conventional Commits** (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`).
5. Configure a Node project safely: a `.gitignore` that covers the standard noise and secrets,
   an `.env.example` instead of a committed `.env`, and a `.nvmrc` pinning the cohort's Node
   version.
6. State, in your own words, the difference between `git revert` and `git reset`, and between
   merge and rebase.

## Prerequisites

| Need | Detail |
| --- | --- |
| Accounts | A **free GitHub account** |
| Tools | VS Code, Git 2.x, a Node version manager (**fnm** used here; nvm/n as alternatives), Bruno (optional today) |
| Cohort pin | **Node 24 LTS** — this module's `.nvmrc` contains `24` |
| Time | 4 hours (Day 1): ~30 min setup, ~2 h build, ~30 min verify, ~1 h write-up |
| Knowledge | Basic programming concepts. No prior Git or Node required. |
| Check first | `git --version`, `gh auth status` (logged in), `fnm list` (shows a v24.x) |

> **Verify your versions before starting:**
> ```bash
> git --version          # git version 2.x
> gh auth status           # Logged in to github.com ...
> fnm list                 # * v24.21.0  (or any v24.x)
> ```

---

## Key concepts

### 1. The Git mental model: three stores

```
   working tree          staging area            commit
  (files on disk)  -->  (your proposal)  -->  (permanent snapshot)
      git add               git commit
```

- **Working tree** — the actual files you edit.
- **Staging area (index)** — `git add` copies chosen changes here: "include this in the next
  commit." You can stage part of a file and leave the rest.
- **Commit** — `git commit` seals the staged snapshot into history permanently. Commits are
  *snapshots with a parent*, not diffs — which is why `git log` is a walkable graph.

Everyday commands: `git status` (what's where), `git diff` (unstaged) / `git diff --staged`
(staged), `git log --oneline --graph` (history), `git restore <file>` (throw away working-tree
edits), `git stash` (park uncommitted work temporarily).

### 2. Branching, merging, conflicts

A branch is a movable label on a commit — cheap. `main` stays releasable; work happens on
feature branches. When two branches change the **same lines** of the same file, Git cannot
guess which wins and stops with a **conflict**, writing markers into the file:

```
<<<<<<< HEAD                ← your current branch's version
  return 'Hello, Node.js!';
=======                    ← the incoming branch's version
  return 'Hi from Node!';
>>>>>>> conflict/m01-alt-greeting
```

Resolving means deleting the markers and keeping (or combining) the content you want, then
`git add` + commit. Escape hatch: `git merge --abort`.

### 3. Merge vs rebase (conceptual)

- **Merge** adds a commit with two parents — it *records* that two lines of history diverged
  and rejoined. Nothing is rewritten.
- **Rebase** replays your commits on top of another branch — linear, prettier history, but the
  commits become *new* commits (new hashes). Never rebase anything already pushed and shared.
- For this module's deliverable — a resolved conflict **visible in history** — merging shows
  more. `gh pr merge --merge` (merge commit) instead of `--squash` is deliberate: squash would
  erase the resolution.

### 4. GitHub Flow & pull requests

1. Branch from `main` → 2. Commit (Conventional Commits) → 3. Push → 4. Open a PR →
5. Review (a partner, or self-review solo) → 6. Merge. One PR = one unit of work.

### 5. Conventional Commits

```
<type>: <lowercase summary>
```
Types you'll use daily: `feat` (new behaviour), `fix` (bug), `docs` (documentation only),
`refactor` (no behaviour change), `chore` (tooling/housekeeping). The DoD needs **3 or more**
on the merged PR.

### 6. Node project hygiene

- **`.gitignore`** — keeps generated/secret files out of history: `node_modules/`, `.env`,
  `*.db`, the generated Prisma client, `dist/`, `coverage/`. A nested `.gitignore` applies only
  to its own directory — that's how this shared repo keeps modules independent.
- **`.env.example`** — commit the *shape* of your config, never the values. `.env` holds real
  secrets and must never be committed — not once, not "just for now" (history is forever).
- **`.nvmrc`** — one line, `24`, so `fnm use` / `nvm use` selects the cohort's Node version.

### 7. `git revert` vs `git reset` (short version — you must be able to narrate this)

| | `git revert` | `git reset` |
| --- | --- | --- |
| What it does | Creates a **new commit** that undoes an older one | **Moves the branch pointer** backwards |
| History | Only grows — safe for shared branches | Rewrites — old commits drop off the branch |
| Variants | (none needed) | `--soft` (stays staged), `--mixed` (default, back to working tree), `--hard` (discarded) |
| Use when | Anything pushed / shared | Local cleanup of commits nobody has |

Rule of thumb: **shared → revert, private → reset**, and check `git status` before `--hard`.

---

## Step-by-step demonstrations

Run these inside the shared repository. If it doesn't exist yet, step 0 creates it once —
reuse it for every future module.

### Step 0 — One-time: the shared repository

```bash
gh auth status                                  # confirm login
mkdir node-js-training-ground && cd node-js-training-ground
git init -b main                                # empty repo, default branch = main
# create README.md (what this repo is for), then:
git add README.md
git commit -m "docs: add shared training ground readme"
gh repo create juzzthynperez/node-js-training-ground --public --source . --push
```
*Why public:* branch protection on **private** repos depends on the GitHub plan; a public repo
avoids that dependency. This is the program-wide convention.

### Step 1 — Scaffold the module directory on `main`

```bash
mkdir -p modules/m01-dev-environment-git-github
cd modules/m01-dev-environment-git-github
```

Create three files (full contents shown in the write-up):

- `.gitignore` — `node_modules/`, `.env`, `*.db`, `prisma/generated/`, `dist/`, `coverage/`
- `.env.example` — `NODE_ENV=development`, `PORT=3000`
- `.nvmrc` — `24`

```bash
git add modules/m01-dev-environment-git-github/
git commit -m "chore: scaffold m01 module directory"
git push
```

### Step 2 — Branch before you type

```bash
git checkout -b feat/m01-hello-node
```

Create `hello-node.js`:

```js
console.log('Hello, Node!');
```

```bash
git add modules/m01-dev-environment-git-github/hello-node.js
git commit -m "feat: add hello-node script"
```

### Step 3 — Keep committing (build your 3+ Conventional Commits)

Edit the script to extract a function:

```js
function greeting() {
  return 'Hello, Node!';
}

console.log(greeting());
```
```bash
git commit -am "refactor: extract greeting message into a function"
```
Add the module `README.md` with run instructions:
```bash
git add modules/m01-dev-environment-git-github/README.md
git commit -m "docs: add run instructions to module readme"
```
Check where you stand:
```bash
git log --oneline        # feat / refactor / docs so far
git status               # clean before you branch again
```

### Step 4 — Deliberately create a conflict

```bash
git checkout -b conflict/m01-alt-greeting
# edit the return line to:  return 'Hi from Node!';
git commit -am "feat: use casual greeting"

git checkout feat/m01-hello-node
# edit the SAME line to:    return 'Hello, Node.js!';
git commit -am "fix: use full product name in greeting"

git merge conflict/m01-alt-greeting
# CONFLICT (content): Merge conflict in .../hello-node.js
```

### Step 5 — Resolve it by hand

Open the file, delete the three marker lines, keep the line you want:

```js
function greeting() {
  return 'Hello, Node.js!';
}
```

```bash
git add modules/m01-dev-environment-git-github/hello-node.js
git commit --no-edit -m "chore: resolve greeting-line conflict"
git log --oneline --graph      # two branches, one merge node — the evidence
```

### Step 6 — Push and open the PR

```bash
git push -u origin feat/m01-hello-node
gh pr create --title "M01: hello-node script with resolved merge conflict" \
             --body "5 Conventional Commits; conflict resolved in history."
```

### Step 7 — Merge (never squash here)

```bash
gh pr merge 1 --merge --delete-branch     # --merge keeps the full history
gh pr view 1 --json state                 # -> "MERGED"
```

### Step 8 — Verify on `main`

```bash
git checkout main && git pull
git log --oneline --graph                 # resolution commit still visible
fnm exec --using 24 -- node modules/m01-dev-environment-git-github/hello-node.js
# -> Hello, Node.js!
git log --all -- "**/.env"                # empty = no secret ever committed
```

---

## Hands-on exercises

All exercises run inside `modules/m01-dev-environment-git-github/` of the shared
`node-js-training-ground` repository. Budget ~2 hours total.

**Exercise 1 — Toolchain & pin (15 min).** Install/confirm VS Code, Git, fnm, Bruno. Create
`.nvmrc` with `24`, run `fnm use && node --version`, and confirm you see a `v24.x`. Add the
scaffold files and commit with `chore:`.
*Done when:* `node --version` inside the module shows v24 and the scaffold commit is on `main`.

**Exercise 2 — The everyday commands (20 min).** Make a small edit to `hello-node.js`, then
walk the loop without looking up: `git status` → `git diff` → `git add` → `git diff --staged`
→ `git commit`. Then break something on purpose and recover it with `git restore`. Finally
`git stash` an edit, run `git stash pop`, and confirm the edit returned.
*Done when:* you can say what `restore` and `stash` each did, unprompted.

**Exercise 3 — Branch → PR (30 min).** Create `feat/<your-initials>-hello-tweak`, change the
greeting, and land **three** Conventional Commits (e.g. `feat:`, `docs:`, `refactor:`). Push
and open the PR with `gh pr create`.
*Done when:* the PR exists on GitHub showing 3+ conventional commits.

**Exercise 4 — The conflict (40 min).** Reproduce Step 4–5: two of your own branches editing
the same line, `git merge`, resolve by hand, commit the merge. Then answer: which marker told
you which side was "yours"? (`<<<<<<< HEAD`.)
*Done when:* `git log --oneline --graph` on your PR branch shows the merge node, and
`node hello-node.js` still runs.

**Exercise 5 — Hygiene audit (15 min).** Intentionally `touch .env`, `git status` (is it
ignored?), then `git add -f .env && git reset .env` — confirm it never reaches a commit.
Check every path in the brief's list is covered by `.gitignore`.
*Done when:* `git log --all -- "**/.env"` is empty and every ignore entry exists.

**Exercise 6 — Narrate (20 min).** Out loud, no notes: `revert` vs `reset`, then merge vs
rebase. Record yourself or explain to a partner.
*Done when:* you complete both explanations without pausing to look something up.

---

## Verification checks

Run these before declaring the module done. Each maps to a Definition-of-Done item.

- [ ] `gh pr view <n> --json state` → `"MERGED"`, and the PR shows **3 or more** Conventional
      Commits (`git log --oneline main..HEAD` counts them).
- [ ] `git log --oneline --graph` shows a **merge node** — the resolved conflict is in history.
- [ ] `git ls-tree -r main --name-only | grep m01` lists `.gitignore`, `.env.example`.
- [ ] `.gitignore` covers all six: `node_modules`, `.env`, `*.db`, Prisma client, `dist`,
      `coverage`.
- [ ] `git log --all -- "**/.env"` returns **nothing**.
- [ ] `fnm exec --using 24 -- node .../hello-node.js` prints `Hello, Node.js!` and
      `node --version` shows v24.
- [ ] You can narrate `revert` vs `reset` out loud, unaided (Exercise 6).
- [ ] Write-up sections filled in as you went — not retro-fitted at the end.

---

## Common mistakes

| Mistake | Why it hurts | Instead |
| --- | --- | --- |
| Committing straight to `main` | No review, history fragments | Branch first — always |
| `git add .` from the repo root | Sweeps other modules' and secrets' files in | Stage explicit paths |
| Message like "changes" / "update" | No history to read later | Conventional Commits: `feat: ...` |
| Committing `.env` "just once" | History is forever; secrets leak | `.env.example` only, real `.env` ignored |
| Squash-merging this PR | Erases the conflict-resolution evidence | `gh pr merge --merge` |
| Letting the editor resolve markers for you | You never learned to read a conflict | Resolve by hand, read all three marker lines |
| Assuming local `node` is v24 | Runs the wrong version silently | Check `.nvmrc`; `fnm use` / `fnm exec --using 24` |
| `git reset --hard` while things are odd | Discards uncommitted work permanently | `git status` + `git stash` first |
| Writing the write-up at the end | Thin, forgets the real pitfalls | Capture as you go |
| One giant PR | Unreviewable | One unit of work per PR |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `fatal: not a git repository` | Wrong directory | `cd` to the repo root; `git status` to confirm |
| `Authentication failed` on push | gh/token/credential issue | `gh auth status`, re-login: `gh auth login`; or check remote URL (`git remote -v`) |
| `CONFLICT (content)` and you're stuck | Normal during a merge | Read markers, keep the wanted lines, or bail out: `git merge --abort` |
| `hint: You have divergent branches` | Local and remote both moved | `git pull --rebase` (solo) or merge, then push |
| `rejected ... non-fast-forward` | Remote has commits you lack | `git pull` first, resolve if asked, then `git push` |
| `node: command not found` after `fnm use` | fnm env not loaded in this shell | Use `fnm exec --using 24 -- node ...`, or re-open the terminal |
| Wrong Node version | Default ≠ `.nvmrc` pin | `fnm use` inside the module dir; confirm `node --version` = v24 |
| `warning: LF will be replaced by CRLF` | Windows `core.autocrlf` | Cosmetic; note it, don't chase it |
| `.env` shows in `git status` | `.gitignore` missing/misspelled | Check the entry exists in the *same directory*; it must never be staged |
| PR merged but history looks flat | Squashed by default | Merge with `--merge`; reopen the PR flow if evidence is needed |
| `gh: command not found` | CLI not installed | Install GitHub CLI, then `gh auth login` |
| `npm ERR!` / native build failures | Missing build tools | Install them now (they block Module 7 onward) — out of scope today, but don't skip |

---

## Knowledge checks

Answers in the key below — try first, unaided.

1. In one sentence each: what do `git add` and `git commit` move, and between which stores?
2. Your working tree has two edited files; you want only one in the next commit. What do you do?
3. What three lines of text appear in a conflicted file, and what does each mean?
4. You resolved the conflict. Which two commands finish the merge?
5. Write a valid Conventional Commit for: a new CLI flag; a documentation-only fix; removing
   dead code with no behaviour change.
6. The PR needs 3 commits but has 2. The DoD also mentions squash merges. Which merge strategy
   do you use, and why?
7. A commit you already pushed to `main` introduced a bug. `revert` or `reset`? Why?
8. You committed a real `.env` locally but have **not** pushed. What are your options?
9. `git log --oneline --graph` shows a commit with two lines joining beneath it. What is that
   commit, and what does it prove for this module?
10. `fnm list` shows v22 as default and v24 installed; `.nvmrc` says `24`. What must you run
    before `node hello-node.js`, and what should `node --version` print?

**Answer key**
1. `add` copies working-tree changes into the staging area; `commit` seals the staged snapshot
   into permanent history.
2. Stage only that file: `git add path/to/file` (never `git add .` from the root).
3. `<<<<<<< HEAD` = your branch's version; `=======` divides; `>>>>>>> <name>` = incoming
   branch's version. Delete all three, keep the content you want.
4. `git add <resolved file>` then `git commit --no-edit` (or `-m "chore: resolve ..."`).
5. `feat: add --verbose flag` / `docs: clarify install steps` / `refactor: remove dead code`.
6. **`--merge`** — squash collapses the branch into one commit and erases the conflict
   resolution, which *is* the deliverable.
7. `revert` — it adds an inverse commit; `reset` would rewrite pushed history and break
   anyone's clone based on it.
8. Delete/replace `.env`, `git rm --cached .env`, commit the removal — still local, so history
   hasn't leaked; better, never commit it and use `.env.example`. (If it *had* been pushed:
   treat the secret as burned and rotate it.)
9. A **merge commit** (two parents). It proves a conflict was merged and resolved in this
   branch's history.
10. `fnm use` (or `fnm exec --using 24 -- node ...`); expect `v24.x`.

---

## Learner checklist

**Setup**
- [ ] Read `brief.md` fully before starting
- [ ] Toolchain ready: VS Code, Git, fnm, Bruno
- [ ] `gh auth status` shows a logged-in free GitHub account
- [ ] `.nvmrc` created with `24`; `fnm use` + `node --version` shows v24
- [ ] Shared repo `node-js-training-ground` exists and is pushed (created once, reused)

**Build**
- [ ] All everyday commands practised: `init`/`clone`, `status`, `add`, `commit`, `log`, `diff`,
      `restore`, `stash`
- [ ] `revert` vs `reset` looked up (not guessed) and explained in my own words
- [ ] `.gitignore` covers all six Node-project entries
- [ ] `.env.example` present; real `.env` never committed
- [ ] `hello-node.js` written on a feature branch — never directly on `main`
- [ ] GitHub Flow looked up; PR opened from the feature branch
- [ ] Merge vs rebase understood conceptually
- [ ] Conflict deliberately created between two branches and resolved by hand
- [ ] PR merged with 3 or more Conventional Commits, using a merge commit (not squash)

**Verify**
- [ ] PR state is `MERGED` with 3+ Conventional Commits and a visible merge node
- [ ] `.gitignore` and `.env.example` confirmed tracked on `main`
- [ ] `git log --all -- "**/.env"` is empty
- [ ] Script runs on Node 24 and prints `Hello, Node.js!`
- [ ] `revert` vs `reset` narrated out loud, unaided
- [ ] Every Definition-of-Done box in `brief.md` genuinely checked

**Write-up & understanding**
- [ ] Write-up filled in as I went (personal reflections there — nothing invented)
- [ ] Checkpoint evidence captured: PR link, `.gitignore`/`.env.example`, revert-vs-reset
- [ ] Knowledge checks 1–10 attempted before reading the key; missed ones re-studied
- [ ] Can teach branch → PR → merge to the next trainee using my own example

**Confirm understanding** — I am confident I can, without looking up:
- [ ] Explain working tree → staging → commit with an example
- [ ] Create a branch, commit, push, open and merge a PR
- [ ] Read conflict markers and resolve a conflict by hand
- [ ] Say when to `revert` vs `reset`, and when to merge vs rebase
- [ ] Explain why `.env` never goes in the repo but `.env.example` does
