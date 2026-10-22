# Patterns — discovered during M08 (2026-10-21)

> Uncommitted draft written by the assistant. rnd git (commit/push/PR/merge) stays with the owner.

## Probe the composition you ship, not the component — Zod decides key-optionality from the field's top-level type

- The list-query field pipeline (`unknown → .refine(repeat) → .pipe(preprocess → inner.optional())`)
  passed every standalone probe — including `field.safeParse(undefined)` → success — and then
  400'd on **every absent query param** once used as an object property: `Invalid input:
  expected nonoptional, received undefined`. Zod's object parser asks the field's *top-level*
  type "may this key be missing?", and a bare pipe isn't `optional` no matter what sits inside
  it. The first Bruno run failed 6/23; the fix was one outer `.optional()` on the pipeline.
- Generalize: schema libraries resolve composition rules at *each nesting level*, and a probe
  that exercises the leaf says nothing about the object's view of it. The standing probe cases
  for any validation library are the shapes real traffic produces — *absent key inside the
  object*, *empty object through the derived schema*, *repeated key* — not just the happy leaf.
  Bruno (integration) caught what schema-level unit probes (component) provably could not.

## A default is a creation concept — `.partial()` over `.default()` resurrects it

- Deriving the update schema as `createTaskSchema.partial()` made `PATCH {}` parse as
  `{ status: "todo" }`: `.partial()` relaxes *presence*, it does not subtract the sibling's
  `.default()`. The phantom field satisfied the cross-field "at least one" refine and the empty
  patch answered **200 instead of 400**. Both schemas now derive from a default-free base
  (`taskFields(…)` factory + bare `statusField`); only `createTaskSchema` adds `.default("todo")`.
- Generalize: derivation operators compose forward, never by subtraction — `.pick`/`.omit`/
  `.partial` transform the *shape* they're given, including every behavior-carrying modifier
  attached to a field. When an endpoint's semantics differ (PATCH must not invent defaults),
  the shared base must be built at the *lowest common denominator* of those semantics, and each
  endpoint adds its own delta.

## Wire-visible strings are contract; shared logic may parameterize presentation

- m07 froze **different wording per context** for the same fields — create: `Required,
  non-empty string` / `Must be a string when present`; PATCH: `Must be a non-empty string` /
  `Must be a string`. A single pretty constant for the shared field vocabulary silently
  changed two wire strings that no Bruno request asserts. The fix keeps logic single-source and
  wording byte-identical: `taskFields({ title, description })` factory whose only parameter is
  the context message pair.
- Generalize (extends M07's contract-fidelity review): the frozen suite is a *floor*, and the
  unasserted remainder is exactly where "cleanup" drifts silently. The check that finds it is
  cheap and mechanical — diff the old layer's *message literals* (grep the string constants out
  of the deleted file) against the new layer's, case by case, before writing schemas. Run it
  FIRST, as a spec input, not last as a fix (this one was found late and cost a re-verification
  cycle).

## Zod 4's unified `error` parameter has one stable form: the string

- On 4.6.7, `{ error: { message: "…" } }` **crashes** when a `.refine()` error is read
  (`iss.inst?._zod.def?.error is not a function` inside `finalizeIssue`); the string form
  `{ error: "…" }` and the legacy `{ message: "…" }` work everywhere. Every schema in the build
  uses the string form; the crash and both survivors are captured in `evidence/m08/zod4-probe.ts`
  / `zod4-api-check.txt`.
- Generalize: the M07 lesson ("remembered behavior is a hypothesis; settle it against the
  installed binary") repeats for library *API surface*, not just CLI behavior — especially in a
  major version's early minors. The habit that pays: a 30-line probe script run against the
  pinned version *before* the schema file exists, testing every customization form you plan to
  use (here: string param, object param, legacy param, on a plain schema AND on `.refine`).

## `unrecognized_keys` is one issue with a keys[] payload — expand it where the envelope is built

- Zod reports unknown keys as a single issue whose `path` is the **object's** path and whose
  payload is `keys: ["foo"]`. Mapped naively you emit `{ path: "", message: 'Unrecognized key:
  "foo"' }` — wrong path, wrong wording, losing m07's one-detail-per-key behavior
  (`{ path: "foo", message: "Unknown field" }`). And a schema-level `error` on the object hijacks
  that issue's message too, so the wording cannot be fixed in the schema — it lives in the
  central handler's `flatMap`, hardcoded, next to the envelope it serves.
- Generalize: library error *shapes* that aggregate (unknown keys, union failures, superRefine
  batches) need an explicit mapping decision per shape — "one detail per actionable item, at the
  path the old layer used". Diffing the old validator's output cases against the new mapper,
  case by case, is the same restore-or-document method as M07's behavior review, one layer down.

## Destructive-by-default tooling has a non-destructive twin — prefer it, especially under AI guards

- Prisma 7.10.x's consent guard correctly blocked `migrate reset` from agent context. The target
  was a freshly-created empty `dev.db`, so the recorded-history path did the identical job with
  no consent surface: `db:deploy` (replays the two recorded migrations) + `db:seed`.
- Generalize: for agents, "rebuild from recorded history" is a strictly better verb than
  "reset and re-provision" whenever the history is complete — it is reviewable, non-destructive
  by construction, and consent-free. Reserve resets (with explicit human consent) for the case
  history itself must be rewritten. The M07 consent env-var note still applies when reset is
  genuinely the right tool.

## A `.refine()` on a strict object runs even when the parse failed — restore mutual exclusivity with a pre-parse guard

- Zod 4 runs an object `.refine()` against the *dirty* parsed value, so a body `.strict()`
  already rejected with `unrecognized_keys` also collects the refinement's issue:
  `PATCH {"foo":1}` emitted `{path:"foo","Unknown field"}` **and** a spurious
  `{path:"","At least one of …"}`. m07 made the two classes mutually exclusive with an early
  return (`if (errors.length > 0) return { errors }` sat *before* the emptiness check) — a
  semantic the Zod composition didn't reproduce, and no Bruno request asserted the difference
  (the /apply envelope capture even *labeled* the two-detail output "m07 parity"). The fix
  moves the cross-field rule ahead of the parse:
  `z.unknown().refine(isPlainObject-nonempty).pipe(z.object(…).partial().strict())` — guard
  fails → only its whole-object issue (the pipe stops); guard passes → only the body's own
  issues. The predicate skips non-objects so arrays and strings still fall through to the
  object's "Body must be a JSON object" instead of being misread as empty patches.
- Generalize: refinements are not transactional with the parse that precedes them — they run on
  whatever value the parse produced, *including partial and failed* values. When the old layer's
  error classes were mutually exclusive (early returns), the replacement must re-create the
  exclusivity structurally (guard-then-parse, or explicit issue filtering), not assume the
  library collects issues the way the old if-chains did. And the parity check is a *behavior*
  diff of the old layer — a capture of the new layer labeled "parity" proves nothing about the
  old one; the pre-PR review (via a subagent with fresh eyes) is what caught both this drift
  and the mislabel.

## Coercion widens the accepted grammar — re-create the old gate on the raw shape, before it

- `z.coerce.number()` runs `Number()` on whatever it sees, so `?page=007`, `1e3`, `+1`, `0x10`
  — all 400s under m07's `/^[1-9][0-9]*$/` — parsed fine and answered **200**. The fix
  re-creates m07's gate as a string-first stage and pipes into the coercion:
  `z.string().regex(/^[1-9][0-9]*$/).pipe(z.coerce.number<string>().int().min(1))`, every
  stage carrying the field's one frozen message (the `<string>` generic pins the coerce's input
  type so the pipe typechecks; a bare coerce declares `unknown` and won't).
- Generalize: a hand-written validator's regex/`parseInt` gate is *two* behaviors fused — shape
  acceptance and value parsing. A coercion API collapses them into one and silently widens the
  first; parity requires re-splitting them. Probe with the shapes real clients accidentally
  send (leading zeros, scientific notation, `+` signs, hex), not just obvious junk.
