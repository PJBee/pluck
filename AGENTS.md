# AGENTS.md

Guidance for AI agents (and humans) working in this repo. Read this before making
changes. Keep it current: if a rule here stops matching reality, update it.

## What this project is

`pluck` — a personal command-line tool for curating a music library at
`/Volumes/Data/Music`, organized as `Category/Artist/Album (YYYY)/tracks`. Two
categories drive everything:

- **Core** — hand-picked individual tracks.
- **Core Album Fills** — the full albums behind those Core picks.

The tool moves tracks between the two (**promote** up into Core, **demote** down
into Fills), **dedupes** Fills against Core, and does general **tag cleanup**.
Matching is on embedded **title tags** (via `ffprobe`), normalized — never on
filenames.

## Workflow — act like a careful developer

- **Plan before non-trivial work.** Before implementing any feature, fix, or
  refactor of real substance, write a new plan file under `planning/`, named
  `NN-plan-name.md` — lowercase, dash-separated, where `NN` is the next number in
  sequence (zero-padded, e.g. `01-build-pluck-package.md`, `02-add-rescan-cache.md`).
  It states the intent, the approach, and any trade-offs. Pause for review before
  writing code — the point is that a human can check your intent before you commit
  to it. Small, obvious changes (typos, comment fixes, tightening a test) don't
  need a plan entry — use judgment.
- **Plans are committed history.** Plan files live in the repo and are committed
  alongside the code they describe — they're reviewable in PRs and preserved in
  `git` history. Treat them as append-only: a plan records what was decided *at
  that time*, so create a new numbered file per effort rather than rewriting old
  ones. If a decision changes, write a new plan; you may add a `Status:` line to
  an existing plan (e.g. `Status: implemented in #14`, `Status: superseded by 03`)
  so a reader knows where it stands. Keep genuinely throwaway scratch out of the
  repo — put it under `planning/scratch/` (gitignored), not in a numbered plan.
- **Conventional Commits.** After any substantial change, commit using
  [Conventional Commits](https://www.conventionalcommits.org): `feat:`, `fix:`,
  `refactor:`, `test:`, `docs:`, `chore:`, etc. The body explains *why*, not just
  *what*. Keep commits focused — one logical change each. Example:

  ```
  feat(promote): match destination album ignoring trailing (YYYY)

  Fills rips are sometimes tagged with a different year than the Core
  album folder (e.g. Core "The Black Parade (2006)" vs a 2010 rip), so
  match album folders on the title alone, case-insensitively.
  ```

- **Pull requests when it's worth review.** Open a PR when a change spans
  multiple commits, alters observable behavior, or is otherwise worth a look
  before merge. Skip the ceremony for trivial one-offs. Never merge your own PR
  without being asked.
- **Don't commit broken work.** Commit only when the change is complete and the
  test suite passes. Never `git push` or open a PR unless asked or clearly
  expected.
- **Branch, don't work on the default branch** for anything non-trivial.

## Safety — hard invariants (do not violate)

This tool moves and deletes files in a real, irreplaceable music library. These
are not preferences:

- **Dry-run by default.** Every command that moves or deletes must default to a
  preview and require an explicit `--apply` (or equivalent) to touch the disk.
- **Trash, never `rm`.** "Deletions" go to the macOS Trash (recoverable),
  including matching `.lrc` sidecars.
- **Never overwrite.** A move must never clobber an existing destination file —
  report and skip instead.
- **Never operate on the real library in tests or development.** Use synthetic
  trees in a temp dir (see Testing).

## Code conventions

- **Python 3.13** target; no legacy version floor (personal tool, single machine).
- **Layered design.** Domain logic (ffprobe/tag reading, normalization, library
  model, moves/dedupe) lives in a reusable core with no surprise side effects.
  Commands and UIs are thin layers on top. Don't duplicate domain logic across
  commands.
- **Keep the TUI and any server stdlib-only** (no third-party deps in those
  layers). The CLI layer may use Typer.
- Preserve the hard-won matching logic when touching it: title-tag extraction,
  the `norm()` normalization (smart quotes/dashes, lowercasing, stripping
  trailing version qualifiers like "remaster"/"live"/"acoustic"/"radio edit"),
  and album matching that ignores the trailing ` (YYYY)`. Don't "improve" these
  casually — they encode real edge cases. If you find a genuine bug, note it and
  fix it deliberately, with a test.

## Testing

- Use `pytest`.
- Build a **synthetic library tree in a tmp dir** with **stubbed/monkeypatched
  ffprobe output**. Tests must never read from or write to `/Volumes/Data/Music`.
- Cover the core hardest: `norm()` edge cases, album matching ignoring `(YYYY)`,
  dedupe selection, and the no-overwrite / sidecar-move rules.
- Run the full suite before every commit.

## External tools

Runtime prerequisites are external binaries, not pip deps — assume they're
installed and document them in the README:

- `ffprobe` (from ffmpeg) — reading tags.
- `kid3-cli` — writing tags (tag cleanup).
