# Patterns — discovered during M04 (2026-10-19)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Express the invariant in the type, not in an assertion

- `noUncheckedIndexedAccess` broke the M02 `mapLimit` body at its core read:
  `items[index]` is `T | undefined`, and `index < items.length` does **not** narrow it —
  the compiler has no algebra relating the guard to the access.
- Three escape hatches, ranked: `items[index]!` (an invisible promise), a `!== undefined`
  guard (silently *skips* legitimately-undefined elements — a real behavior bug, not a
  workaround), or restructure so the type carries the guarantee. The restructure: a
  shared `Symbol.iterator` pull, where `step.done === false` narrows `step.value` to `T`
  by definition. Same semantics, order preservation intact, zero assertions.
- Teaching form: this is the module's thesis in one edit. When a strict option fights
  you, the option has found a place where your code's *claim* isn't expressed — fix the
  claim's expression, don't silence the checker. Standing pattern for M05+: any
  `!`/`as` added to shut up `noUncheckedIndexedAccess` is a review smell.

## Definite-assignment (TS2454) exempts `unknown` — validate the claim, don't seed around a ghost

- Draft claim, written confidently: `let lastError: unknown;` used after a `for` loop
  → "TS2454 waiting to happen", so seed `= undefined`. Validation called it: a minimal
  repro under TS 7.0.2 `--strict` compiles clean (exit 0) — **TS2454 never fires for
  `unknown`** because `undefined` inhabits `unknown`, so "possibly unassigned" is
  in-type. The same pattern with `let lastError: string` *is* flagged (TS2454) — the
  exemption is about the type, not the loop.
- The seed had masked the question for the whole build: nothing ever *tested* the
  comment's claim. Seed removed, comment rewritten to the verified behavior. (The loop
  genuinely always runs — `retries >= 0` is checked first — so a real error is always
  captured; that's a separate, behavioral reason, not a compiler one.)
- Teaching form: a compiler-behavior statement in a comment deserves the same
  evidence standard as a code claim — a three-line repro, two `tsc` runs (positive and
  negative control: `string` fires, `unknown` doesn't). "Believed claim + masking
  seed" is the failure mode; the negative control is what makes the finding
  trustworthy.

## assert.rejects validators are `unknown` too — narrow first

- First typecheck of the port failed on one error: TS18046 reading `.name` inside
  `assert.rejects(..., (error) => …)` — the validator parameter is `unknown`, same class
  of problem as `catch (e)`. Reordering `assert.ok(error instanceof Error)` (an
  assertion signature, so it narrows) before member reads fixed it.
- Captured the planted-error run in evidence (`sample-compiler-error.txt`) — a *real*
  compiler message with a fix order is better teaching material than a contrived one.
- Recurring shape: any callback receiving "a value from elsewhere" (validators, DOM
  handlers, express middleware params) is `unknown` under strict — the port should teach
  the *recurring* pattern, not the one-off.

## erasableSyntaxOnly is the bridge config — it's what makes native stripping honest

- `erasableSyntaxOnly` in tsconfig does double duty: it's the compile-time gate that
  guarantees `node file.ts` *can* run the file (no enums/namespaces/parameter
  properties to choke on), so the awareness-level native attempt on Node 24 can't fail
  for syntax reasons the config already forbade.
- The demo run needs explicit `.ts` import extensions (`allowImportingTsExtensions`,
  legal because `noEmit`) — same extension Node's stripper and tsx both require. One
  config choice, two runners satisfied.
- Pattern: pick config options that make the *weakest* runtime path (native stripping)
  succeed by construction, then let the strongest gate (`tsc --noEmit`) carry correctness.

## Node 24.21.0 native type stripping: worked clean, no warning — but scope the claim

- `node async-toolkit.ts` → full 16/16 self-checks, exit 0, **no experimental warning
  printed** on v24.21.0. Tempting to write "stripping is stable on 24" — but the brief
  pins full stability to Node 26, and a warning-free run on one small file is one data
  point, not a support matrix. Write-up scoped it as "worked today on a file this small"
  vs "is stable" — same honesty pattern as M03's signal-handling claim.

## fnm exec spawn trap on Windows — third manifestation, now fully characterized

- The v22-default probe (known since M01) plus a NEW failure: `fnm exec --using=24 -- npm
  …` dies with "Can't spawn program: program not found" — fnm's direct exec doesn't
  resolve the Windows npm shim. `node` inside exec works fine; only the npm/npx shims
  fail.
- Working form (used for every command this module): `fnm exec --using=24 -- bash -c 'npm
  run typecheck'`. Now captured alongside the version probe as the standing session
  opener for M05+. The `.envrc`/shell-level fix backlog item should include the shim
  problem, not just the default-version problem.

## Trainee material: config-before-code ordering beats concept-before-config — for setup

- M04's section order within "setup": `package.json` must exist *before*
  `npm install --save-dev` (or `--save-dev` has nowhere to record), scripts declared
  before the files they run exist (scripts are declarations; `typecheck` works
  immediately, `toolkit` works once Section 7 lands). Two forward-references, both
  smoothed with one line each instead of reordering the whole module.
- Check for this class when linting: every command that *writes state* (install, init)
  must come after the file that state is written into, and before anything that reads
  it.

## Flaky timing self-checks: clock granularity, not a broken port — but fix anyway

- A re-run after the TS2454 fix produced 15/16: `resolves after the requested delay`
  failed with `expected >= 50ms, waited 49ms` — `Date.now()` deltas can land ~1ms under
  a correctly-waited `sleep(50)`. Three immediate re-runs passed; it's measurement
  granularity, not timer misbehavior.
- Kept anyway as FIX NOW: in *teaching* material a 1-in-N flake reads as "my port is
  broken" — trainees have no baseline to distinguish noise from regression. Lower-bound
  timing asserts get a documented tolerance (`>= 45` for a 50ms sleep; `>= 95` for two
  50ms delays) with a comment naming the real failure mode being guarded (setTimeout
  clamp → ~0-5ms), which the tolerance still catches by an order of magnitude.
- Verify the fix under load, not once: 6 consecutive full runs (tsx + native) after the
  change. Pattern generalizes: assert on *order of magnitude and direction*, not on the
  exact wall-clock number you saw the one time you measured.

## Solution-manual drift check now has a mechanical test

- Section 7 and Section 9 (the two lab sections) grep for `^function`/`^async function`
  at line start must return 0 complete bodies — requirements, done-when criteria, and
  signature-only fragments are the allowed forms. Ran it; passed. Cheaper than reading.

## DoD removal: convert assessment apparatus into Checkpoint seams (2026-10-20 revision)

- The beginner-friendly revision deleted the formal Definition of Done (4 lists, 8
  `- [ ]` boxes) and the `evidence/m04/` trainer dossier — but *verification itself was
  never removed*, only relocated: each "Done when:" became a bolded **Checkpoint:**
  line inside the exercise it gates, §1 got one "How you'll know you're there" paragraph
  naming the three proofs, and §9/§10 ended on back-to-back gate runs instead of a
  review checklist. Post-state invariants: zero `Definition of Done`, zero `- [ ]`, zero
  `evidence/` instructions, exactly 5 `**Checkpoint:**` (one per old "Done when").
- Teaching form: assessment removal in trainee material is a *reshaping* operation, not
  a deletion — grep for the apparatus words before/after; if the verification commands
  (`npm run typecheck`, `npm run toolkit`) also disappeared, the cut went too far.

## Plan line-targets lose to the scope floor — flag, don't dig (2026-10-20)

- The plan promised ≈950–1,050 lines (from 1,212); the honest end state was 1,122. The
  last ~70 lines were reachable only by cutting brief-scope content (tsconfig
  essentials, utility-type examples, lab requirements). Rule applied: the scope list is
  the acceptance criterion, the line target was an estimate — stop trimming at the
  floor, ship, and state the deviation in APPLY COMPLETE rather than silently
  under-delivering content or silently overshooting the plan.
- Compression that *did* work without content loss: a 30-line option-bullet walkthrough
  → a 7-row option table (~-20 lines, escape pipes as `\|`); folding the standalone
  "tsc build workflow" subsection into the gate subsection's lead sentence; dropping
  `Parameters`/`FirstArg` (not in brief's utility list).
