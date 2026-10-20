# Patterns — discovered during M02 (2026-10-16)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Consistent failure surfaces beat per-path correctness

- The first self-check run failed 14/15 because `sleep` threw its `RangeError` *synchronously*
  while `retry`/`mapLimit` (async functions) rejected. `assert.rejects` needs a promise — a
  sync-throwing path is untestable by the same assertion as the others.
- Rule: in a utility module, make **every** failure arrive the same way (rejections for async
  functions). A tool that sometimes throws and sometimes rejects forces callers to handle two
  shapes — that inconsistency is a bug even when each path works.
- Teaching form: "same door for every error."

## Self-check entry-point guard (ESM, no test framework)

- Checks must run under `node file.js` but NOT when another module imports a utility.
  Working guard: `import.meta.main ?? (path.resolve(process.argv[1]) === fileURLToPath(import.meta.url))`.
- Getting it backwards fails silently in both directions: checks re-run on every import, or
  never run at all while output looks "successful".

## Re-applying a reverted merge (GitHub ancestry trap)

- GitHub creates PRs by *ancestry*: after `git revert -m 1 <merge>`, the original branch's commits
  are already in main's history → "No commits between main and <branch>", PR impossible.
- Fix: `git rebase --onto main <pre-merge-base> <branch>` (replays the commits on top of the
  revert; clean when main's tree equals the old base), `git push --force-with-lease`, then PR.

## Probe the version manager, not the config file

- `.nvmrc` says 24; a fresh shell answered `v22.23.2` — fnm does not auto-apply the pin here.
  Every run used `fnm exec --using=24 -- node …`. Same M01 lesson, new costume: a checklist item
  is done only when the tool *answered*, not when the config file promises it.

## Timing measurements that survive review

- Warm up with one throwaway request (DNS/TLS cost otherwise poisons the first run).
- Parallelism proof: a single slow sequential run (2236ms vs 37ms parallel) is the clearest
  evidence that sequential *sums* latencies while parallel *overlaps* them — record every run,
  report medians, keep the outlier as the anecdote.
