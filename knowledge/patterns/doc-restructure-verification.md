# Patterns — verifying large document restructures (2026-10-19)

> Discovered while re-outlining `modules/m02-.../training-material.md` (16 sections → 10)
> in the rnd repo. Uncommitted draft; rnd git stays with the owner.

## Backup first, then prove coverage with a line-level `comm` diff

- Before overwriting a big doc, copy it to temp. After the rewrite, extract "content"
  lines (≥30 chars; skip headings, fences, `---`), `sort -u` both sets, then
  `LC_ALL=C comm -23 orig_set new_set`. Every "missing" line must be classified as an
  *intentional* change (ref rewrites, rewraps, merged steps) or an accidental drop.
- Per-line `grep -F` in a loop is correct but far too slow — it timed out at 120s on a
  1370-line file on this box. The set-difference is one pass and finishes in seconds.
- This converts "I think I moved everything" into a reviewable list. In the M02 run it
  surfaced a reference the renumbering itself had broken: a signpost I rewrote as
  "back to §8" during the move, where the original (and correct) wording was
  "back to this section" — under the new numbering §8 means a different topic entirely.
  The flagged line made the mis-point obvious; it plus two whitespace/wording drifts
  were the only real defects — exactly the class a re-outline produces silently.

## Renumbering breaks inline section cross-refs — audit in the same pass

- `grep -n '§[0-9]*'` (or `§N`, "section N") **after** writing, and verify each hit
  resolves to the intended target under the new numbering. A re-outline can move a
  reference's *meaning* without changing its text (§8 used to mean "promise
  coordination", now it means "async mistakes") — the digit may even be absent
  ("this section"). Check semantics, not just digits.

## Chunked full rewrites with an APPEND-MARKER

- A full-file rewrite that exceeds one comfortable output: `write` chunk 1 ending with a
  unique marker (e.g. `<!-- APPEND-MARKER -->`), then `edit` marker → chunk N + marker,
  final edit removes it. Finish with `grep -c APPEND-MARKER` = 0 to prove no scaffolding
  survived.
- When the old section order is preserved by the new outline (contiguous group merges),
  map old→new sections first — you almost never need to move content across the file,
  only retitle/merge headings. The move list stays small and reviewable.

## Verify structure with cheap invariants, not a re-read

- Heading lint: one `##` per section, numbers 1…N in order, subsection labels match the
  approved outline exactly; fence count even; expected separator count (`---` = N−1);
  numbered-step ranges per section. Seconds to run, catches most structural regressions.

## Anchors in edit spans: repeat them verbatim or keep them out (2026-10-21, M06)

- A body rewrite whose `oldString` starts at a heading must repeat that heading in
  `newString` — one M06 edit dropped `## 7. Querying Relational Data` exactly this way
  (oldString spanned heading + bullets + opener; newString was the new opener only). Caught
  in-pass, but a heading loss survives every content diff — it is the one line the coverage
  classifier never flags as a drop.
- Cheap defense: keep structural anchors (headings, fence delimiters) OUT of edit spans
  where possible — anchor on unique body text instead; when the anchor must be in the span,
  diff the heading set (`grep '^## '`) immediately after the edit batch, not at end of pass.

## glob cannot see gitignored state dirs — detect tool backends with the shell (2026-10-21)

- `/evaluate`'s "check which backend exists" step (glob `graphify-out/graph.json`) silently
  reports NO GRAPH when one exists: glob skips gitignored paths, and `graphify-out/` is
  gitignored. The graph was present the whole session; `ls graphify-out/` found it instantly.
  Any existence probe for tool-generated state must use the shell, not glob.
- Related: `graphify path` remains unusable on this graph (ambiguous basename match), and
  `graphify query` resolves the bare name `training-material.md` to M01's node — six modules
  share that filename, so basename node identity shadows other modules' files. Query with
  module-qualified strings and read the `src=` attribute to confirm which file a node is.

## A checkpoint's numbers are document-wide: grep the old value after an outline swap (2026-10-26, M06 rev)

- The owner's new outline changed §9's checkpoint from "evidence of both outcomes" to
  "all three outcomes." The 9 checkpoint swaps passed char-for-char — but the *same count*
  still lived in three other places (intro deliverables list, §9's two prose paragraphs,
  §10's DoD checklist), stating 2 where the contract now says 3. Headings and checkpoints
  are gated; the prose that *agrees with* a checkpoint is not.
- Cheap version: after any outline edit that adds, drops, or renumbers a quantity
  ("both"→"three", "20+"→"25+", "1:N"→"1:N and 1:1"), `grep -i` the old value across the
  whole file before closing the pass — the stale copies are always outside the edit span.
- Generalization of M05's two-renderings class: the unit of consistency is not the edited
  line, it's every line that *asserts the same fact*.
