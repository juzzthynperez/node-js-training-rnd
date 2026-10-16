# Patterns — discovered during M01 (2026-10-16)

## Verify tools by how they actually answer, not how a guide says to
- Version managers differ per machine: this Windows box uses **fnm**, not nvm. Probing the
  wrong name (`nvm: command not found`) is not a broken install.
- Desktop apps may have no CLI on PATH out of the box: **Bruno's desktop app doesn't
  bundle one** — verify via `winget list Bruno` / installed-apps, or by opening it. Bruno
  *does* ship a separate CLI (`npm i -g @usebruno/cli` → `bru`) for those who prefer the
  terminal — flavor is a preference; what matters is verifying *your* install actually works.
- A checklist item is done only when the tool *answered* — probe, don't assume.

## `git restore` reads from the index, not from HEAD
- Staged **new** file + `git restore <file>` = file **stays** (index holds the staged copy).
  Tracked edited file + `git restore <file>` = edits discarded (index holds the old version).
- Proof technique: `git hash-object <file>` == `git rev-parse HEAD:<file>` after restore.
- Teaching form: "restore = copy from a room; the flag decides which room."

## Ignore-rule-first for trainer-private evidence
- Commit `evidence/` in the `.gitignore` **in the root commit**, before any note exists.
  Rules added after the near-miss are rules that already nearly failed.
- Public checkpoint trail = PR/merge history + committed files; private notes quote into
  write-ups inline, never via committed paths.
