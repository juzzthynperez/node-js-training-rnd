# M04 Tasks — TypeScript Basics & TypeScript on Node.js

Read and write strict TypeScript, and set up a TypeScript project that runs and type-checks on
Node.js.

> There is no solutions file for this stage. This checklist guides the work; it doesn't contain it.

## Setup

- [x] Read this stage's `brief.md` fully; have your M02 toolkit code and write-up, plus your M03
      write-up, on hand.
- [x] Confirm your local Node version matches the cohort's pinned Node 24 LTS (`.nvmrc`) and install
      `@types/node@24`.
- [x] Continue the branch → PR → merge habit for this module.

## Research — core TypeScript

- [x] Why TypeScript exists, structural typing, and inference vs explicit annotation.
- [x] Primitives, arrays, tuples, unions, intersections, literal types, and `as const`.
- [x] `type` vs `interface`, optional and `readonly` properties, and why this course uses union
      types instead of `enum`.
- [x] Functions and generics (function generics, constraints, defaults).
- [x] Narrowing techniques: `typeof`, `in`, equality checks, discriminated unions, user-defined type
      guards.
- [x] `unknown` vs `any` vs `never`.
- [x] The utility types `Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`,
      `Awaited`.
- [x] `satisfies` and `import type`.

## Research — TypeScript on Node.js

- [x] Project setup: installing TypeScript 7 and matching `@types/node` to Node 24.
- [x] `tsconfig.json` essentials: `strict`, `module`/`moduleResolution: nodenext`,
      `verbatimModuleSyntax`, `erasableSyntaxOnly`, `noUncheckedIndexedAccess`, `noEmit`,
      `skipLibCheck`.
- [x] The three ways to run TypeScript — `tsx`, Node's native type stripping, and a `tsc` build
      (awareness only) — and what each is for.
- [x] `tsc --noEmit` as the type-check gate, wired up as an `npm run typecheck` script.
- [x] Typing async code (`Promise<T>`), typing caught errors (`catch (e: unknown)`), typing
      `process.env`, and what `@types/*` packages and `.d.ts` files are for.
- [x] What changed from TypeScript 6 to 7 (strict by default, legacy options removed,
      native-compiler speed) and what the missing programmatic API means for tooling — you'll meet
      this again in M09.
- [x] Practice reading a few real compiler errors without panicking, before you need to under lab
      pressure.

## Build

- [x] Set up the TypeScript project (`tsconfig.json`, scripts) before porting any code.
- [x] Port the M02 toolkit's `retry` to TypeScript as `retry<T>`.
- [x] Port `mapLimit` to TypeScript as `mapLimit<T, R>`; port `sleep` and `withTimeout` too.
- [x] Model the Task domain: a `Task` type, then derive `CreateTaskInput` and `UpdateTaskInput` from
      it using utility types rather than writing them by hand.
- [x] Model a discriminated-union `Result<T, E>`.
- [x] Run the ported toolkit with `tsx`.
- [x] Attempt running it with Node's native type stripping, and note what actually happens on
      Node 24.

## Verify

- [x] Zero errors from `tsc --noEmit` on a strict config.
- [x] You can explain `unknown` vs `any`, and `type` vs `interface`, unaided.
- [x] The toolkit runs successfully via `tsx`.
- [x] You've attempted native type stripping on Node 24 and documented what actually happened
      (awareness-level on this Node version, not a hard pass/fail requirement).
- [x] Final self-review against every Definition of done checkbox in `brief.md`.

## Write-up

- [x] "What I built".
- [x] "Why it's built this way (key decisions)": which utility types you used for
      `CreateTaskInput`/`UpdateTaskInput` and why, how you modeled `Result<T, E>`, and what
      specifically happened when you attempted native type stripping on Node 24.
- [x] "How to build it (teach it to the next trainee)": write the utility-types guide.
- [x] "Concepts worth explaining": pick 1-2 ideas and explain each in your own words.
- [x] "What tripped me up": compiler errors that confused you at first, and the difference between
      `tsx` and native type stripping on Node 24.
- [x] "Checkpoint evidence": zero `tsc --noEmit` errors on your strict config, the toolkit running
      via `tsx`, and what happened when you attempted native type stripping on Node 24.
- [x] Close out "What I'd do differently".
