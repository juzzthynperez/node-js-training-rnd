# Patterns — Git / shared training repository

Discovered during M01 APPLY/VALIDATE. Verified in `juzzthynperez/node-js-training-ground`.

## Sequence a merge-conflict demo *after* the shared line exists

Conflict branches must be created from a commit where the target line already exists (e.g.
after the refactor that turns `console.log(...)` into `return '...'`). Branching earlier means
the two sides edit different lines — Git auto-merges silently and the learner gets no conflict.
Observed rule in `training-material.md`: *branch for the conflict after the refactor.*

## One shared repo, nested `.gitignore` per module

In the shared `node-js-training-ground` repo, each module directory carries its own
`.gitignore`. Git applies a nested `.gitignore` only to its own subtree, so modules (and their
`node_modules`, `.env`, `*.db`, Prisma client, `dist`, `coverage` rules) stay independent
without a repo-per-module. Also: never create a separate public repo for a module unless a
genuine requirement (CI secrets, publishing rights) is identified and explained first.

## Merge, never squash, when the conflict resolution *is* the deliverable

`gh pr merge <n> --merge --delete-branch`. Squash collapses the branch into one commit and
erases the conflict-resolution merge node from history — destroying the evidence the Definition
of Done asks for. Always verify afterwards with `git log --oneline --graph`.

## Verified one-shot pinned Node run (fnm)

`fnm use` only affects a shell with fnm's env loaded. For a guaranteed run under the cohort pin
from any shell: `fnm exec --using 24 -- node <script>` (observed: prints `v24.21.0`).

## No-secrets verification

After any push to the shared public repo: `git log --all --oneline -- "**/.env"` must be empty.
Public repos are world-visible and indexed — a pushed secret is burned and must be rotated;
history rewrite does not un-leak it.
