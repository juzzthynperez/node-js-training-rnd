# M01 — Dev Environment, Git & GitHub

**Beginner-level training material** · Week 1, Day 1 · Study time: ~4 hours
**Companion docs:** [brief](brief.md) · [tasks](tasks.md) · trainer's [write-up](write-up.md)

---

## 1. Welcome to Your Development Environment

### What You Will Learn

- How the tools of Node.js development fit together: a code editor, Git, Node.js itself, and a version manager
- How to write and run your first Node.js program
- What version control is, and the everyday Git workflow every developer uses
- How branches, pull requests, and GitHub let people build software together
- The habits of a clean, safe, professional repository

### What You Will Build

A small but complete foundation, published on GitHub:

- A working development environment you've verified yourself
- A `hello-node` program that runs on your machine
- A public repository with a proper `.gitignore` and `.env.example`
- **A merged pull request containing at least 3 Conventional Commits and a real merge conflict you planted and resolved yourself**

That last item is the heart of this module. Anyone can read about merge conflicts; you will *have* one, fix it, and merge anyway.

### Prerequisites

- A computer with an internet connection (Windows, macOS, or Linux)
- A free [GitHub](https://github.com) account
- Willingness to type commands in a terminal — everything here is typed, nothing is clicked-only

No programming experience beyond "code is text that a computer runs" is assumed.

### Tools You Will Use

| Tool | What it is | Why you need it |
| --- | --- | --- |
| **VS Code** | A free code editor from Microsoft | Where you write and read code |
| **Git** | A version control program, run in the terminal | Remembers every change you make and lets you undo, compare, and combine work |
| **Node.js** | A runtime that executes JavaScript outside a browser | Runs your programs |
| **A version manager** (nvm / nvm-windows / fnm) | A small tool that installs and switches Node versions | Lets you (and your team) agree on one Node version per project |
| **GitHub** | A website that hosts Git repositories | Where your code lives so others can see and review it |
| **A terminal** | A text window where you type commands | Where Git and Node actually run |

### How to Approach the Hands-On Exercises

1. **Read the section first, then do the exercise.** The exercises in §10 repeat what you just read — that repetition is deliberate.
2. **Type the commands yourself.** Copy-pasting works but teaches your fingers nothing; typing builds memory.
3. **Run things and read the output.** Every command here prints something useful. If output surprises you, that's the lesson.
4. **It's hard to break anything.** Git records history — mistakes are recoverable (you'll learn exactly how). Experiments are the point.
5. **Get stuck for a minute, then use §11 (Troubleshooting), then ask.** Struggling briefly is how the knowledge sticks.

---

## 2. Setting Up Your Development Workspace

### Understanding the Tools Used in Node.js Development

A Node.js project is usually touched by four kinds of tool at once: an **editor** (VS Code) for writing code, a **runtime** (Node.js) for executing it, **Git** for tracking changes, and **GitHub** for sharing them. They're independent — you can install them in any order — but they only *feel* like one system once you've used them together, which is exactly what this module makes you do.

### Setting Up Your Code Editor

1. Download VS Code from [https://code.visualstudio.com](https://code.visualstudio.com).
2. Install it and open it once. You'll see a **Welcome** tab — you can close it.
3. Open a folder via **File → Open Folder…** — this becomes your project's home.

Two habits now, so they're automatic later: VS Code has a built-in terminal (**Terminal → New Terminal**), and `` Ctrl+` `` (the backtick key, top-left of the keyboard) toggles it. You'll use it constantly.

### Installing Git

- **Windows:** download from [https://git-scm.com](https://git-scm.com). Accept the defaults except: choose **"Git from the command line and also from 3rd-party software"**, and for line endings pick **"Checkout Windows-style, commit Unix-style"**.
- **macOS:** installing Git prompts you to install Apple's developer tools — accept, or run `git --version` in Terminal and follow the prompt.
- **Linux:** `sudo apt install git` (Debian/Ubuntu) or your distribution's equivalent.

Verify:

```bash
git --version
# git version 2.x.x
```

Then introduce yourself — Git stamps this on every commit you make:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Use **the same email as your GitHub account** (GitHub lists your commits under it; Settings → Emails shows your verified addresses).

### Installing Node.js

Download the **LTS** (Long Term Support) version from [https://nodejs.org](https://nodejs.org). "LTS" means stable and supported — the choice for every real project. **However** — see the next section; most developers install Node *through a version manager* instead of directly, because that's how you keep several versions without conflict.

Verify whichever you installed:

```bash
node --version   # e.g. v24.x.x
npm --version    # comes with Node; you'll use it in M03
```

### Managing Node Versions

Projects pin different Node versions. A **version manager** installs any version on demand and switches between them in one command:

| Platform | Version manager | Install | Use |
| --- | --- | --- | --- |
| macOS / Linux | **nvm** | [github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm) | `nvm install 24`, `nvm use 24` |
| Windows | **nvm-windows** or **fnm** | [github.com/coreybutler/nvm-windows](https://github.com/coreybutler/nvm-windows) | `nvm install 24`, `nvm use 24` |

(Or [fnm](https://github.com/Schniz/fnm) everywhere — it's faster and works the same way.)

Two caveats if you're on Windows:

- **nvm-windows may need the exact version, not the short form.** If `nvm use 24` answers
  with *"Could not find or select"*, run `nvm list` to see your installed versions and use
  the full one it shows (e.g. `nvm use 24.21.0`). fnm and macOS/Linux nvm accept the short
  form — this is a nvm-windows quirk, not a broken install.
- **Check `nvm list` (or `fnm list`) after installing.** Seeing Node 24 in the list proves
  the install worked, whichever tool you chose.

Install the version this program pins (check with your trainer; current cohort: **Node 24 LTS**):

```bash
nvm install 24
nvm use 24
node --version    # confirm it changed
```

### Understanding Project-Level Node Versions

A version manager's real power: each project can **declare** its Node version in a one-line file called **`.nvmrc`**:

```bash
echo 24 > .nvmrc
```

Now, inside that project's folder, `nvm use` (no argument) reads `.nvmrc` and switches to the pinned version automatically. The whole team uses the same Node version without anyone having to remember it — the file remembers.

**Windows caveat — don't trust the redirect blindly.** `echo 24 > .nvmrc` behaves slightly differently across Windows shells: in **cmd.exe** it writes `24` *with a trailing space*, and in **PowerShell** it can write the file in UTF-16 encoding with a BOM. Some version managers then fail to parse the pin. After creating the file, always look at it:

```bash
cat .nvmrc        # should print exactly: 24
```

If yours shows `24 ` (trailing space) or unreadable characters, just create the file in VS Code instead and type `24` on one line — that's the safe, shell-independent way.

**Critical distinction:** creating `.nvmrc` does *not* switch your current version. It's a declaration; the version manager acts on it. Always confirm with `node --version` after `nvm use`.

### Getting Comfortable with the Terminal

You need five terminal skills for everything in this module:

| Command | What it does |
| --- | --- |
| `pwd` (or `cd` on Windows) | "Where am I?" — print the current folder |
| `ls` (or `dir` on Windows) | "What's here?" — list the folder's contents |
| `cd my-folder` | Move into a folder |
| `cd ..` | Move up one folder |
| `Tab` | Autocomplete the path you're typing — **use this constantly** |

Two more truths: commands are case-sensitive (`Git` ≠ `git`), and the folder you're *in* matters — Git and Node act on the current directory. "I ran the command and it did nothing" is usually "I ran it in the wrong folder."

### Verifying Your Setup

Run these — every line should print a version, not an error:

```bash
git --version
node --version
fnm --version       # your version manager (nvm / fnm / nvm-windows — whichever you installed)
code --version      # VS Code, if its command-line tool is installed
pwd                 # you exist somewhere sensible
```

Two real-world quirks worth knowing before they surprise you:

- **Your version manager may not be the one in the guide.** The tables above list nvm and
  fnm as the common choices; whichever one you installed is *yours* — verify with its name,
  not with whatever a tutorial mentions. (`nvm: command not found` when you installed fnm
  is not a broken install, just the wrong name.) Also confirm it actually *manages* the
  cohort version: `fnm list` / `nvm ls` should show Node 24 installed, not just your
  current default.
- **Bruno comes in two flavors — pick either.** The **desktop app** (what this program's
  exercises use — install from [bruno.com](https://www.usebruno.com) or your package
  manager, e.g. `winget install Bruno.Bruno` on Windows) doesn't bundle a command-line
  tool, so there's nothing to `--version` from a terminal — verify it by opening the app,
  or check your installed-apps list. If you *prefer* working from the terminal, Bruno also
  ships an official CLI as a separate install (`npm install -g @usebruno/cli`, which gives
  you the `bru` command — `bru --version` then works). Same tool, two doors: the desktop
  app for clicking through requests, the CLI for scripting and CI (you'll see the CLI's
  value in M09's CI module). Neither install is "more correct" — it's a preference.

✅ **Checkpoint:** every command that *can* print a version does, and Bruno opens. Do
**Exercise 1** in §10.

---

## 3. Your First Node.js Program

### Understanding How Node.js Runs JavaScript

Browsers run JavaScript inside web pages. **Node.js runs the same language on your computer**, directly — no browser, no HTML. This is what makes JavaScript a tool for backends, scripts, and command-line programs. When you type `node something.js`, Node reads that file, executes it top to bottom, and prints whatever the program outputs.

### Creating Your First Node.js File

In VS Code, open your project folder, create a file named `hello-node.js`, and type:

```js
// hello-node.js — first Node.js program
const name = "Node.js trainee";

console.log(`Hello from ${name}!`);
console.log(`Running on Node ${process.version}`);
```

Two things are happening here that deserve a second look:

- `` `...${name}...` `` is a **template literal** — backticks let you embed variables (or any expression) inside a string with `${...}`.
- `process.version` is Node telling you its own version — e.g. `v24.1.0`. Every program that will run on someone else's machine benefits from printing this.

### Running a JavaScript File

In the terminal, move into the folder containing the file, then:

```bash
node hello-node.js
```

Expected output:

```
Hello from Node.js trainee!
Running on Node <your version>
```

That's it — that's a Node.js program. (No build step, no compilation, no server required.)

### Understanding Standard Output and Console Messages

`console.log(...)` writes a line to **standard output** — the text your terminal displays. It's the oldest debugging tool in existence and still the first one you'll reach for. Try it: add `console.log("testing 123");` anywhere in the file, run again, and watch it appear. (You'll remove it in the next section — that's also the point.)

### Making and Testing a Small Change

A developer's smallest complete loop is **edit → run → look**:

1. Change the greeting to something else.
2. Run `node hello-node.js`.
3. Read the output. Does it match what you expected?

Do it once now. You'll repeat this loop thousands of times; getting the rhythm early makes every later module easier.

### Exploring the Project Directory

Your project so far:

```
your-project/
├── hello-node.js     ← the program
└── .nvmrc            ← (if you made one in §2) the pinned Node version
```

Note the dot-prefixed `.nvmrc`: **files starting with `.` are hidden** in most terminals (`ls -a` shows them). Git and Node both use dotfiles for configuration — you'll meet several.

✅ **Checkpoint:** you can explain, out loud, what `node hello-node.js` actually does. Do **Exercise 2** in §10.

---

## 4. Understanding Git and Your Code's History

### Why Developers Use Version Control

Imagine keeping `hello-node.js`, then `hello-node-v2.js`, `hello-node-FINAL.js`, `hello-node-FINAL-real.js`… Everyone who has worked this way has lived this nightmare. Version control replaces the copies with **history**: every change is recorded, labeled, and recoverable. Delete something important last Tuesday? It's in Tuesday's commit. Want to see what changed this morning? One command. Want two people editing the same file without destroying each other's work? Branches (§7).

### What Git Does

**Git** is a program on your machine that keeps a **repository** (`.git` folder) full of snapshots of your project. It answers four questions forever:

1. What did the project look like at any point in history?
2. What exactly changed between two points?
3. Who changed this, when, and why (via the message)?
4. How do I combine two lines of work safely?

### Git vs. GitHub

The single most common beginner question, so be precise:

- **Git** is the *local* program — the history lives on your computer, in the project's `.git` directory. It works offline.
- **GitHub** is a *website* that stores copies of Git repositories so people can share and collaborate. Many others exist (GitLab, Bitbucket); "GitHub" is to Git repositories what "YouTube" is to videos — the best-known host, not the technology itself.

**Git works perfectly without GitHub.** Everything in §4 and §5 happens before any network is involved.

### Understanding a Git Repository

`git init` turns any folder into a repository — it creates a hidden `.git/` directory. That folder *is* Git: delete it (don't) and you have plain files with no history. Everything else you'll type is Git reading or adding to `.git`.

### Understanding the Working Directory

The **working directory** (or working tree) is simply *your files as they currently are on disk* — the text in VS Code right now. Nothing is recorded yet. Edit a file and it's "modified in the working tree."

### Understanding the Staging Area

The **staging area** is a middle room where you *choose* what goes into the next snapshot. `git add file.js` copies the file's current state into it. Why a middle room? Because a commit should be one coherent change — if you edited three files but only two belong to one idea, you stage those two. The staging area lets you compose commits instead of dumping everything.

### Understanding Commits

A **commit** is a permanent snapshot of the staged files plus a message, given an ID (a hash like `e5a5d1f`) and linked to its parent. Together they form a **chain** — the history. Commits are cheap: make many small ones. A history of five focused commits teaches a reviewer (or your future self) what happened; one giant "stuff" commit teaches nothing.

### Visualizing How Changes Become Part of History

The same file moves through three rooms:

```
   EDIT              git add             git commit
WORKING DIR  ──────────────►  STAGING AREA  ──────────────►  COMMIT (history)
 (your files)                 (next snapshot)                (permanent)
```

- `git status` tells you **which room each file is in** — learn to read it as a sentence ("modified: hello-node.js" = it differs from the last commit; "Changes to be committed" = it's staged).
- `git diff` compares the working dir against staging; `git diff --staged` compares staging against the last commit.

Almost all early Git confusion is losing track of which room a change is in. `git status` is the answer to that question, every time.

✅ **Checkpoint:** narrate the three rooms and what `git add` does, without looking. Then do **Exercise 3** in §10.

---

## 5. Your Everyday Git Workflow

This is the module's core loop — the sequence you will use in every module and every job. It assumes a repository exists (make a practice one now: `git init` inside your project folder).

### Creating or Downloading a Repository

- **Start a new one:** `git init` — turns the current folder into a repository with an empty history on the `main` branch.
- **Copy an existing one:** `git clone <url>` — downloads a repository *with its full history* and sets up a remote called `origin`. This is what you'll use for GitHub-hosted projects.

### Checking the Current State of Your Files

```bash
git status
```

The most-used Git command. Read it every time before and after anything else:

```
On branch main
nothing to commit, working tree clean        ← everything recorded; you're safe

Changes not staged for commit:
        modified:   hello-node.js            ← edited, NOT in the next commit yet

Changes to be committed:
        new file:   hello-node.js            ← staged; the next commit WILL include it
```

### Staging Changes

```bash
git add hello-node.js      # stage one file
git add .                  # stage everything changed in this folder (careful — see §6)
git diff                   # see WHAT you're about to stage, while it's still un-staged
```

Stage deliberately. `git add .` on autopilot eventually commits a secret file you didn't mean to share.

### Committing Changes

```bash
git commit -m "feat: add hello-node script printing a greeting"
```

The commit now contains exactly what was staged — nothing more, nothing less, no matter what else changed on disk. That predictability is why the staging room exists.

### Writing Meaningful Commit Messages

A commit message is a message to a human reading the history in six months. Two parts:

- **A short header line** — imperative mood ("add", "fix", "remove" — not "added" or "I added"), under ~72 characters.
- **Why, when it isn't obvious** — as a body after a blank line: `git commit` with no `-m` opens an editor; write the header, blank line, then the reasons.

"fix stuff" is a message to no one. "fix: read config before starting server — env vars were undefined at import time" saves an hour of future archaeology.

### Reviewing Previous Commits

```bash
git log                    # full history, newest first (q to quit the viewer)
git log --oneline          # one line per commit — the readable overview
git show <commit-id>       # everything one commit changed
```

Run `git log --oneline` now on your practice repo. Seeing the chain makes history feel real.

### Comparing File Changes

```bash
git diff                   # working dir vs staging — "what am I about to stage?"
git diff --staged         # staging vs last commit — "what will the next commit record?"
```

Output convention: lines prefixed `-` are removed, `+` are added. If you ever wanted to know *exactly* what someone changed, this is how.

### Undoing Uncommitted Changes

**Discard one file's uncommitted edits** (back to the last commit — this destroys the edits, so be sure):

```bash
git restore hello-node.js
```

**Unstage a file** (keep the edits on disk, remove from the staging room):

```bash
git restore --staged hello-node.js
```

The `--staged` flag is the whole difference: "take it out of staging" vs "throw away the edits."

One subtlety that catches almost everyone: `git restore <file>` restores **from the staging
area**, so it behaves differently depending on where the file currently lives:

- **Tracked file with uncommitted edits** — the staging area holds the last-committed
  version, so `git restore` puts the file back to it. The edits are gone. (You can *prove*
  it worked: `git hash-object <file>` will equal `git rev-parse HEAD:<file>`.)
- **New file you just staged** (`git add`ed for the first time) — the staging area holds
  *that new file*, so `git restore` restores the staged copy: the file **stays**. To make a
  staged new file disappear entirely, take it out of staging first
  (`git restore --staged <file>`, then delete it from disk).

The mental model, one last time: "restore" always means *copy from a room* — with no flag,
that room is the staging area, not the last commit. When a restore "didn't work," check
which room the version you expected actually lives in.

### Temporarily Setting Changes Aside

Halfway through a change, need a clean tree for something urgent? `git stash`:

```bash
git stash                    # sweep uncommitted changes into a drawer; tree is clean
git stash pop                # bring them back
git stash list               # see all drawers
```

The stash is a holding pen, not a commit — nothing is in history and nothing is lost, but nothing is *saved* either. Pop it the same day.

### Choosing the Right Way to Undo a Change

Commits are where undoing gets serious, and there are two very different tools:

| | `git revert <commit>` | `git reset <commit>` |
| --- | --- | --- |
| **What it does** | Makes a **new commit** whose content *undoes* the given commit | **Moves the branch pointer** back — later commits fall off the branch |
| **History** | Grows; nothing rewritten | Rewritten; "undone" commits are no longer on the branch |
| **Safety** | **Safe on shared branches** — nobody's history changes | Only safe for **your own unpushed** commits |
| **Recovery** | Just another commit | Reflog remembers for a while — recoverable, but don't count on strangers knowing that |

**Rule of thumb:** pushed/shared branch → `revert`. Your own branch, unpushed mistake → `reset` is fine (usually `--hard`, which also discards the working-dir edits). When in doubt, `revert` — it's never wrong, only more ceremonial.

✅ **Checkpoint:** the full loop — status → edit → diff → add → commit → log — without prompting. Do **Exercises 3 and 4** in §10.

---

## 6. Organizing and Protecting a Node.js Project

### Understanding Project Files and Folders

A minimal Node.js project:

```
my-project/
├── .git/            ← Git's history store (never edit directly)
├── .gitignore       ← what Git should NOT track
├── .nvmrc           ← pinned Node version
├── .env             ← real secrets (NEVER committed)
├── .env.example     ← template of .env's keys, committed
├── README.md        ← what this project is
├── package.json     ← project identity (M03)
└── node_modules/    ← installed dependencies (huge; never committed)
```

### What Belongs in Version Control?

**Commit:** source code, configuration *templates* (`.env.example`), `.gitignore`, `.nvmrc`, `README.md`, `package.json` and the lockfile.

**Never commit:** dependencies (`node_modules/`), secrets (`.env`), build output (`dist/`), test coverage (`coverage/`), databases (`*.db`), generated code.

The test: *can this file be recreated by running something?* Then don't commit it. *Does it contain a secret?* Then it must never be committed, ever.

### Understanding `.gitignore`

`.gitignore` is a plain text file listing paths Git should stop watching:

```gitignore
node_modules/
dist/
coverage/
*.db
.env
```

- A trailing `/` means "folders named this."
- `*.db` means "any file ending in .db."
- Git only respects rules for files it **isn't tracking yet** — ignore the file *before* the first `git add`, or see §11 for the fix.

### Ignoring Dependencies and Generated Files

`node_modules/` is the big one — a project's installed dependencies, often thousands of files recreated in one command (`npm install`, M03). Committing it makes repositories enormous and merges impossible. Same logic for compiled output (`dist/`), coverage reports, and Prisma's generated client — **if a machine can regenerate it, Git ignores it.**

### Understanding Environment Variables

**Environment variables** are key-value settings living *outside* your code — read at runtime. In Node, `process.env.API_KEY` reads one. This separates *what the code is* from *the values for this machine*: the same code runs on your laptop, your teammate's, and a server, each with its own values.

### Protecting Secrets and Sensitive Configuration

A **secret** — API key, password, token — pasted into a tracked file is pasted into *history*. Anyone with repo access can find it, and GitHub's scanners will likely flag it within minutes of pushing. Prevention:

1. Keep real values in `.env`, which is **gitignored**.
2. **Never** `git add` a real `.env`. Ever.
3. If one slips into a commit: **rotating the secret** (regenerating the key) is the real fix — removing the file from history doesn't un-leak it. See §11 for the emergency steps.

### Creating an `.env.example` File

If `.env` is never committed, how does anyone know which variables a project needs? The `.env.example` — same keys, placeholder or empty values, **committed**:

```bash
# .env.example — copy to .env and fill in real values. Never commit .env.
DATABASE_URL=
API_KEY=
```

It documents the project's configuration and gives new team members a copy-paste starting point. Even an empty one teaches the habit — commit one in this module's exercise, with nothing in it yet.

### Writing a Useful README

A README answers, for a stranger landing on the repo: **what is this**, **how do I run it**, **how do I use it**. For now:

```markdown
# my-project

One sentence: what it is.

## Run it
    node hello-node.js

## Requirements
Node 24 (see .nvmrc)
```

Imperfect but present beats polished but absent. You'll grow it module by module.

✅ **Checkpoint:** you can list five things that must never be committed, and what `.env.example` is for. Do the hygiene parts of the exercises below.

---

## 7. Working with Branches

### Why Developers Use Branches

If everyone edited `main` directly, every broken half-finished change would land in the shared code. A **branch** is a movable label on a commit — an independent line of work you can develop without touching `main` until it's ready. Branches are what make "work in progress" and "shipped code" coexist.

### Understanding the Main Branch

`main` (called `master` in older repos) is the trunk — the branch considered stable and shared. **Rule for this program, and most teams: never commit directly to `main`.** Everything happens on branches, joins `main` through a pull request (§8).

### Creating and Switching Branches

```bash
git branch feature-greeting      # create a branch (you stay where you are)
git checkout feature-greeting    # switch to it
# or, both at once:
git switch -c feature-greeting   # create AND switch (modern shortcut)
git switch main                  # go back
```

New branches **start at your current commit** with all history — they're cheap, disposable, and free. Make them without ceremony.

### Developing a Feature Separately

With `feature-greeting` checked out, edits, `git add`, and commits go onto *that* branch only. `main` doesn't move. Switch back and forth with `git switch` — your working files change to match the branch each time, because each branch is a different line of snapshots.

### Committing Changes on a Branch

Same loop as §5 — nothing changes except *where* commits land:

```bash
node hello-node.js        # still: run before committing
git add hello-node.js
git commit -m "feat: reword greeting on feature branch"
git log --oneline         # the new commit is on THIS branch only
```

### Bringing Changes Together Through Merging

Finished? Get the work onto `main`:

```bash
git switch main
git merge feature-greeting    # fold the branch's commits into main
```

If nothing diverged, Git does a **fast-forward** — `main`'s label simply moves forward. Clean and invisible. The interesting case is next.

### Understanding Merge Conflicts

When **both branches changed the same lines since they diverged**, Git can't guess your intent — it stops and refuses to merge automatically:

```
Auto-merging hello-node.js
CONFLICT (content): Merge conflict in hello-node.js
Automatic merge failed; fix conflicts and then commit the result.
```

Not an error — a question. Git highlights the problem in the file itself:

```js
<<<<<<< HEAD
console.log(`Welcome from ${name}!`);
=======
console.log(`Hi there from ${name}!`);
>>>>>>> feature-greeting
```

`<<<<<<< HEAD` to `=======` is **your current branch's** version; `=======` to `>>>>>>> feature-greeting` is **the incoming branch's**. `git status` shows the file as `UU` (both modified).

### Resolving a Merge Conflict

1. Open the file. Every marker block is an unanswered question.
2. Edit into **one correct version** — keep one side, the other, or combine them. The result must be *good code*, not just "a side."
3. **Delete every marker line** (`<<<<<<<`, `=======`, `>>>>>>>`) — leftovers break the file confusingly.
4. **Run it.** `node hello-node.js` — a resolved conflict that doesn't run isn't resolved.
5. `git add hello-node.js` — this declares "conflict answered."
6. `git commit` — Git pre-fills a merge message; keep it and add *how* you resolved it, because a reviewer can't guess.
7. `git log --oneline --graph` — admire the fork-and-join shape you just created.

### Understanding Merge vs. Rebase

Two ways to combine branches:

- **`git merge`** creates a new commit joining the two lines, preserving the fork. History shows *when* lines of work met. Nothing is rewritten — **always safe.**
- **`git rebase`** replays your commits one by one onto the tip of the other branch, producing a straight line. Beautifully linear history — but it creates *new* commit IDs, so it **rewrites history**. Golden rule: **never rebase commits other people have pulled.**

Beginner default: **merge.** It can't damage a shared repository, and its history is honest about what happened. (Conceptual here; you'll practice both as your team workflow develops.)

✅ **Checkpoint:** create a branch, commit on it, merge it — and explain why the conflict happened. Do **Exercises 5 and 6** in §10.

---

## 8. Collaborating with GitHub

### Understanding Remote Repositories

So far, everything lived on your machine. A **remote** is a named copy of your repository elsewhere — typically GitHub. One remote per repository is standard, named `origin`.

```bash
git remote -v        # see your remotes (empty until you add one)
```

### Connecting a Local Repository to GitHub

**Option A — start on GitHub, copy locally:**

```bash
git clone https://github.com/you/your-project.git
```

**Option B — local repository already exists:**

1. Create a new empty repository on GitHub (no README — it conflicts with your existing history).
2. Connect and push:

```bash
git remote add origin https://github.com/you/your-project.git
git push -u origin main      # -u links your local main to origin/main
```

After `-u` is set once, plain `git push` / `git pull` just work.

### Authenticating with GitHub

GitHub no longer accepts plain passwords for Git over HTTPS. Pick one:

- **GitHub CLI** (`gh`) — install from [cli.github.com](https://cli.github.com), run `gh auth login`, follow the prompts. Creates a token and stores it; also gives you `gh` commands for PRs (used below). **Recommended — one-time setup.**
- **Git Credential Manager** — bundled with Git for Windows; on first HTTPS push a popup asks you to sign in once, then remembers.
- **SSH keys** — `ssh-keygen -t ed25519`, add the public key in GitHub → Settings → SSH keys, then use `git@github.com:...` URLs. Key-based, no passwords at all.

Whichever you choose, the test is the same: your first `git push` asks for authentication **once** and never again.

### Pushing and Pulling Changes

```bash
git push              # send your local commits to origin (after -u, once)
git pull              # fetch AND merge changes others made into your branch
```

`git pull` before you start work each session — pulling first avoids the painful "my local and the remote both changed" tangles. Pushing publishes; it does not merge.

### Understanding GitHub Issues

**Issues** are GitHub's task list attached to a repository: bugs to fix, features to build, each with a number, discussion, and labels. The habit worth building now: **before you code, open (or reference) an Issue** — "what is this change and why" deserves to exist before the code does. Even solo, an Issue trail makes your repository legible.

### Understanding Pull Requests

A **pull request (PR)** is GitHub's propose-and-review screen: "here's my branch, please merge it into `main`." It shows the commits and every changed line, hosts discussion, runs checks (later modules), and requires an explicit merge. The PR — not the push — is how work enters `main`.

### Following a Collaborative Development Workflow

The full **GitHub Flow**, which you've now assembled piece by piece:

1. `git pull` on `main` — start current.
2. `git switch -c feat/<thing>` — branch off.
3. Work in small commits, running as you go.
4. `git push -u origin feat/<thing>` — publish the branch.
5. Open the PR — describe *what* and *why*.
6. Review (self or teammate), respond, push fixes as new commits.
7. Merge; delete the branch.

### Writing Clear Pull Request Descriptions

A PR description is a favor to your reviewer. Minimal shape:

```markdown
## What
Adds hello-node and the M01 foundation files.

## Why
First module deliverable; demonstrates the branch→PR→merge cycle including a resolved conflict.

## Notes
Conflict between two greeting wordings resolved in <commit> by combining both.
```

Name the tricky parts explicitly — reviewers look where you tell them.

### Reviewing Code and Responding to Feedback

Review reads for correctness, clarity, and safety — line by line, via GitHub's comments. Giving review: be specific and kind ("What happens if this is empty?" not "This is wrong"). Receiving it: every comment is a gift, even clumsy ones — ask if unclear, push fixes as new commits, and reply so the reviewer knows what changed.

### Merging Changes into the Main Branch

Green checks, approval, no unresolved threads → **Merge pull request**. Choose a **merge commit** rather than "Squash" when the history tells a story worth keeping (a squash flattens every branch commit into one and hides it — e.g. the conflict-and-resolution pattern you practiced here). After merging: **delete the branch** — it's merged; the history lives in `main`.

✅ **Checkpoint:** you can recite the 7 steps of GitHub Flow from memory. Do **Exercises 7 and 8** in §10.

---

## 9. Writing Code Like a Team Developer

The previous sections taught Git's commands. This one is about *taste* — the habits that make a repository worth working in. They're all visible in how you do this module's exercises.

### Writing Clear Commit Messages

Covered mechanically in §5; the standard here is **would a stranger understand why this change exists?** Imperative header ("add", "fix", "remove"), body when the *why* isn't obvious from the diff. A commit saying "fix: read config before starting server — env vars were undefined at import time" teaches every future reader. One saying "updates" teaches nothing and costs the same keystrokes.

### Understanding Conventional Commits

A lightweight message format this program (and much of the industry) uses:

```
<type>: <short imperative description>

type ∈ feat | fix | docs | chore | refactor | test | style | perf
```

Examples from this module's own history:

```
chore: add .nvmrc pinning Node 24 LTS and .env.example
feat: add hello-node script printing a greeting and runtime version
fix: resolve greeting conflict by combining both wordings
```

Why it matters: the type is machine-readable (release notes and automation can group changes by kind) and human-scannable (`git log --oneline` becomes a table of contents). Type **before** the colon (`feat`), description **after** it — lower case, imperative, no trailing period.

### Keeping Changes Small and Focused

One commit = one idea. "Add `.nvmrc`" is one commit; ".nvmrc plus reformat the README plus fix a typo" is three. Small commits review in seconds, bisect cleanly when something breaks later, and — as you proved with the conflict — are the unit of safe merging. When you catch yourself staging a fifth unrelated file, that's the signal to commit what you have first.

### Reviewing Your Own Changes Before Sharing

**Before every `git commit`:** `git diff --staged` and actually read it. Before every PR: the full `git diff main...your-branch`. You are the first reviewer, and you catch the embarrassing things cheaply: stray `console.log`, a commented-out block, a secret line, the file you forgot was open. Thirty seconds of self-review saves a colleague spotting it for you.

### Following a Feature Branch Workflow

The pattern from §7–§8, as law: **no direct commits to `main`; everything via branch → PR → merge.** It's not bureaucracy — it's what guarantees every change to the shared branch was at least glanced at by someone (even if that someone is future-you, three days later, on the PR screen).

### Keeping Your Repository Clean and Secure

- `.gitignore` correct **before** files appear, not after they leak.
- Real secrets only in gitignored `.env`; `.env.example` carries the shape.
- Delete merged branches.
- `git pull` at the start of every session; don't let local and remote histories drift apart unknowingly.
- The moment a secret is committed anyway: **rotate the secret first**, then clean history (§11 has the steps) — and tell whoever needs to know. Secrets are only safe once assumed leaked.

✅ **Checkpoint:** the six habits above, in your own words, one sentence each.

---

## 10. Hands-On Practice

All eight exercises run in **one public repository** you create (this program uses a shared `node-js-training-ground` repo — confirm with your trainer). Do them in order; later exercises build on earlier ones. Every "run it" is mandatory — the evidence that exercises worked *is* the running.

### Exercise 1 — Set Up Your Development Workspace

1. Install and verify the §2 toolchain: `git --version`, `node --version`, editor opens.
2. Create and verify a version manager; `nvm install 24` (or your cohort's version), `nvm use 24`.
3. Create a project folder; inside it, `echo 24 > .nvmrc`; run `nvm use` and `node --version` — confirm the pin is honored.
4. `git config --global user.name` / `user.email` set.

**Done when:** every command prints a version and your `.nvmrc` exists.

### Exercise 2 — Create and Run a Node.js Program

1. In your project folder, write `hello-node.js` exactly as in §3.
2. `node hello-node.js` — observe the output.
3. Make one small change (different greeting), run again, observe the diff in output.
4. Add a line printing your name via a template literal; run again.

**Done when:** the script prints your greeting and the *actual* `process.version` of your environment.

### Exercise 3 — Track Changes with Git

1. `git init` (if the folder isn't a repository yet); `git status`.
2. `git add hello-node.js`; `git status` — read the room it names.
3. `git commit -m "feat: add hello-node script"`.
4. `git log --oneline` — see your commit.
5. Edit the file again; `git diff` — read what changed, *without* staging.
6. Restore it: `git restore hello-node.js`; confirm with `git status` and `node hello-node.js`.

**Done when:** you can narrate, for your own file, which room it's in at each step.

### Exercise 4 — Practice Staging, Committing, and Reviewing Changes

1. Make three separate small edits to your project (e.g., tweak the greeting, add a comment, add a print line), staging and committing **each as its own commit** with a Conventional Commit message.
2. Between commits, use `git diff` and `git diff --staged` — know which comparison each shows.
3. Try `git stash`: modify a file, `git stash` (tree clean?), `git stash pop` (edit back?).
4. `git log --oneline` — read your four-or-more commits as a story. Would a stranger follow it?
5. Unstage one file with `git restore --staged <file>` and observe: edits remain on disk.

**Done when:** `git log --oneline` shows several clear Conventional Commits and you've used stash both ways.

### Exercise 5 — Create a Feature Branch

1. `git switch -c feat/hello-tweak` off your current work.
2. Change the greeting message; run it; commit with a Conventional Commit message.
3. `git switch main` — notice the file content reverts; `git switch feat/hello-tweak` — it returns.
4. `git log --oneline --graph --all` — see your two lines of work.

**Done when:** the branch exists, holds one commit, and switching moves your files.

### Exercise 6 — Practice Resolving a Merge Conflict

1. From your feature branch, create two extra branches at the same commit:
   `git branch conflict-a` and `git branch conflict-b`.
2. `git switch conflict-a` → edit **the same line** in one file → commit (`feat: reword … (branch A)`).
3. `git switch conflict-b` → edit **that same line differently** → commit.
4. `git switch conflict-a` → `git merge conflict-b` → **watch the CONFLICT happen.**
5. Resolve per §7: read the markers, produce one correct version, delete all marker lines, **run the file**, `git add`, commit the merge.
6. `git log --oneline --graph` — see the fork and join.

**Done when:** the file runs cleanly post-resolution and the conflict is visible in history.

### Exercise 7 — Publish Your Repository on GitHub

1. Create a new **public** repository on GitHub (empty — no README), or use your trainer's shared training-ground repo.
2. `git remote add origin <url>` (or it exists from `git clone`); `git remote -v`.
3. Authenticate per §8 (GitHub CLI recommended: `gh auth login`).
4. `git push -u origin main` — authenticate once when prompted.
5. Refresh GitHub — your files are there.

**Done when:** `main` with your work is visible on github.com.

### Exercise 8 — Open and Merge a Pull Request

1. Push your feature branch: `git push -u origin feat/hello-tweak` (and the conflict branch you want in the PR).
2. On GitHub: **Compare & pull request**. Base: `main`. Head: your feature branch.
3. Write the description per §8 (What / Why / Notes — mention the conflict and how you resolved it).
4. Confirm the PR shows **at least 3 Conventional Commits** and the conflict-resolution commit.
5. Open the **Commits** tab and the **Files changed** tab — read what a reviewer sees.
6. **Merge pull request** → confirm a merge commit (not squash).
7. `git switch main && git pull` — bring the merge home; verify `node hello-node.js` still runs.

**Done when:** the merged PR link is in your write-up and `main` contains everything.

---

## 11. Troubleshooting Common Beginner Problems

### Terminal Cannot Find Node.js or Git

`'node' is not recognized` (Windows) or `command not found: git` means the program isn't on your **PATH** — the list of folders the shell searches.

- Reinstall Node/Git choosing the option that adds them to PATH (or the "developer tools" prompt on macOS).
- On Windows, the PATH change needs a **new terminal window** — old ones don't see it.
- Quick check: close the terminal entirely, reopen, try again before anything else.

### The Wrong Node.js Version Is Running

You created `.nvmrc`, yet `node --version` shows something else. **`.nvmrc` is a declaration, not a switch.** Run `nvm use` (or `nvm use 24`) *inside the project folder*, then confirm with `node --version`. Also: version managers only affect terminals that were opened *after* installation — restart it. This is exactly why `hello-node.js` prints `process.version`: **make your environment print its own truth.**

### Changes Are Not Appearing in Git

A file edited but invisible to `git status`? Check, in order:

1. **Am I in the right folder?** `pwd` / `git status` from the project root.
2. **Is it ignored?** `git check-ignore -v <file>` prints the `.gitignore` rule responsible.
3. **Is it in a subfolder Git ignores wholesale** (like `node_modules/`)? Intended — keep your code out of there.
4. **Did I actually save the file?** (`Cmd+S` / `Ctrl+S` — it's this more often than anyone admits.)

### Files Were Not Included in a Commit

The commit happened but a file is missing — it was never staged. Good news: it's uncommitted, not lost. Check `git status`; if the file sits unstaged, just commit it now (`git add` + `git commit`). If you staged it but committed something else in between — same answer. Prevention: read `git status` **before** every commit, and treat "Changes to be committed" as the commit's contents, because it is.

### Git Rejects a Push

`! [rejected] non-fast-forward` means the remote has commits you don't — usually because someone (or you, from another machine) pushed. Fix:

```bash
git pull          # integrate the remote's commits (resolve any conflict it reports, as in §7)
git push
```

Prevention: `git pull` at the start of every session. **Never `git push --force` on a shared branch** — it overwrites other people's history.

### GitHub Authentication Fails

- Passwords are **not accepted** for Git operations — you need a token (via GitHub CLI / credential manager) or an SSH key (§8).
- On Windows, an old stored password in Credential Manager will fail repeatedly — update or delete it.
- 403 on push = the token/user lacks permission for that repository — confirm you're pushing to *your* repo, logged in as the right account (`gh auth status`).

### A Merge Conflict Appears

Reassurance first: **a conflict is a question, not a failure.** Steps, again:

1. `git status` — see which files (`UU` = both modified).
2. Open the file; find `<<<<<<<`.
3. Edit into one correct version; delete **all** marker lines.
4. **Run the thing.**
5. `git add <file>` → `git commit` → `git log --oneline --graph` to confirm.

You cannot get stuck here permanently: `git merge --abort` returns everything to before the merge, no harm done. (And `git status` mid-conflict always shows where you are.)

### Sensitive Files Were Accidentally Tracked

Act in this order — the order matters:

1. **Rotate the secret immediately.** If a key/token/password touched a commit, treat it as **leaked** and regenerate it at its source *first*. History cleanup does not un-leak anything.
2. Stop the bleeding: add the file to `.gitignore`; `git rm --cached <file>` (removes from tracking, keeps on disk); commit.
3. If the commit with the secret **was not pushed** — `git reset --soft HEAD~1`, unstage the file, recommit clean (§5's reset, your own branch, safe).
4. If it **was pushed** — the secret is out; rotation (step 1) was the real fix. Rewrite-history tools (`git filter-repo`) exist, but they force-reset everyone's clones — coordinate before using them, and tell your team either way.

The lesson to keep: this whole section is a repair kit for a fire you never want to start. `.gitignore` before the first `git add`. `.env.example`, not `.env`.

---

## 12. Knowledge Check and Self-Assessment

Answer out loud, unaided, then check against the sections. "I'd have to look it up" is a legitimate answer — it tells you *where* to look.

### Development Environment Fundamentals

1. Name the four tool types every Node.js developer uses, and one job of each.
2. Why install Node through a version manager instead of directly from the website?
3. What does `.nvmrc` do — and what does it *not* do by itself?
4. What is the "edit → run → look" loop, and why is running the program before committing non-negotiable?

### Node.js Execution Basics

5. What is the difference between JavaScript in a browser and JavaScript run by Node.js?
6. What does `console.log` actually do?
7. Why does printing `process.version` in a first program earn its place?

### Git Workflow Fundamentals

8. Explain the three rooms (working dir, staging area, commit) and what `git add` does.
9. What's the difference between `git diff` and `git diff --staged`?
10. `git restore file.js` vs `git restore --staged file.js` — what survives each?
11. What does `git stash` do, and is stashed work saved?
12. Someone asks: "Is this change committed?" — what command answers that, and how do you read its output?
13. **Out loud, no peeking:** `git revert` vs `git reset` — what each does, and when each is safe.

### Branching and Merge Conflicts

14. Why do teams forbid direct commits to `main`?
15. What are the two ways to combine branches, and the golden rule about rebase?
16. A merge stops with `CONFLICT (content)`. List your next five actions in order.
17. What do the `<<<<<<<` and `>>>>>>>` lines each represent?

### GitHub Collaboration

18. Git vs. GitHub, precisely, in two sentences.
19. List the seven steps of GitHub Flow from memory.
20. Why doesn't `git push` merge anything?
21. What are three things a good PR description tells a reviewer?
22. Merge commit vs. squash merge — why does this module choose merge commit?

### Repository Hygiene and Secret Management

23. List six patterns a Node.js `.gitignore` must cover, and why each is there.
24. What is `.env.example`, who reads it, and what values go in it?
25. A secret was committed and pushed an hour ago. What is your *first* action, and why isn't it "delete the file"?

### Practical Self-Assessment

The real test — without help, in a fresh folder:

- [ ] Create a repo, write a small Node script, run it, and commit it with a Conventional Commit message
- [ ] Make 3+ commits on a feature branch and explain each one's message
- [ ] Deliberately create a merge conflict between two branches, resolve it, and run the result
- [ ] Publish to GitHub and merge a PR containing that history
- [ ] Narrate `git revert` vs `git reset` and the three Git rooms, unaided
- [ ] Point at your `.gitignore` and explain every line in it

All six checked means M01 is genuinely done. If not, the relevant section is above — that's what it's for.

---

## 13. Wrap-Up

### Key Takeaways

- **The environment declares its requirements** — `.nvmrc` pins Node for everyone; a version manager honors it. Always confirm with `node --version`.
- **Git has three rooms.** Nearly every early confusion is losing track of which room a change is in. `git status` always answers.
- **Small, frequent, Conventional Commits** are the unit of everything good: reviewable, revertable, mergeable.
- **Conflicts are questions, not errors.** Markers in, correct code out, run it, `git add`, commit.
- **Merge is the safe default;** rebase is a rewrite — never on shared history.
- **Branch → PR → merge, every time.** The habit is the course.
- **`.gitignore` first, `.env.example` always, real secrets never.**

### Common Beginner Mistakes to Avoid

- Committing directly to `main` "just this once"
- Staging everything (`git add .`) without reading `git diff --staged` first
- Writing commit messages for the machine ("updates", "stuff") instead of the human reading history later
- Believing `.nvmrc` switches the version by itself
- Committing `node_modules/` or a real `.env`
- Force-pushing to a shared branch
- Avoiding conflicts (aborting, delegating) instead of practicing on cheap throwaway branches

### Development Habits to Practice

`git status` before anything · run before committing · one concern per commit · pull at session start · self-review before every push · delete merged branches · write the PR description *before* opening the PR.

### What You Should Be Able to Do Independently

Take any small change from idea to merged `main` on a public GitHub repository — branch it, commit it with Conventional Commits, open a clear PR, resolve any conflict that arises, merge it — and keep the repository clean and free of secrets while doing it. That is the complete loop this program repeats in every module ahead.

### Preparing for the Next Module

M02 — **Modern JavaScript & Async/Await** — leaves Git basics behind and goes into the language itself: the tools of modern JS, promises, `async`/`await`, and building a small async toolkit. Nothing new to install beyond Node itself; your environment from this module is exactly what you need. Keep the repo — M02's work goes in the same place, same workflow: branch, commit, PR, merge.

---

*Part of the [node-js-training-rnd](https://github.com/juzzthynperez/node-js-training-rnd) training program · Practical work: [node-js-training-ground](https://github.com/juzzthynperez/node-js-training-ground)*
