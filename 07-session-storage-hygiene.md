# Change: Session Storage Hygiene

**Date:** 2026-10-17
**File Modified:** None (documentation only — operational procedure)
**Related:** `05-image-economy-and-visual-diagnosis.md`, `06-three-tier-verification-policy.md`

---

## Summary

Records the one mechanism that actually reclaims space from
`~/.local/share/opencode/opencode.db`, and documents why compaction does not.

---

## The Problem

Playwright screenshots (and any other inline tool images) are stored as base64
in session history inside `opencode.db`.

Observed size at time of writing: **~480 MB**.

This is a *different* store from Playwright's scratch files on disk:

| Store | Grows from | Reclaimed by |
|-------|-----------|--------------|
| `~/.cache/opencode/playwright-screens/` | screenshot files on disk | `--output-max-size` (20 MB budget) automatically |
| `~/.local/share/opencode/opencode.db` | inline images + tool output in session history | **`opencode session delete` only** |

`--output-max-size` has **zero reach** over the database. Confirmed from
`_enforceOutputBudget()`, which only enumerates `outputDir` on disk.

---

## Why Compaction Does Not Help

From the OpenCode compaction docs:

> "**Storage** — Earlier messages remain stored even when they are no longer
> sent to the model."

Compaction replaces what is *sent to the model* with a summary. The underlying
messages — including image payloads — remain in `opencode.db`.

Also from the docs, on the recent conversation kept beside the summary:

> "long tool output is shortened and attachment data is replaced by a short
> description"

That is a **request-time** transformation, not a storage rewrite.

**Do not expect `compaction` settings to shrink the database.**

---

## The Reclaim Lever

`opencode session` provides the only destructive path:

```sh
$ opencode session --help
SUBCOMMANDS
  list      List top-level sessions in the current project, newest first
  delete    Delete a session and its child sessions
  export    Export session data as JSON
  import    Import session data from a JSON file or URL
```

### Routine hygiene

```sh
# 1. See what exists, newest first
opencode session list

# 2. Optionally export anything worth keeping as JSON
opencode session export > ~/Documents/Work/opencode/session-backup.json

# 3. Delete sessions you no longer need
opencode session delete <session-id>
```

`delete` removes a session **and its child sessions**.

> ⚠️ Deletion is permanent. `export` first if there is any doubt — exports are
> JSON and can be re-imported with `opencode session import`.

### Suggested cadence

- **Monthly**, or whenever `opencode.db` grows past a size you find acceptable:
  ```sh
  du -h ~/.local/share/opencode/opencode.db
  ```
- Delete finished/abandoned sessions.
- Keep sessions you still intend to continue.

---

## How This Fits The Overall Design

Storage growth is addressed at three layers:

```
1. PREVENT  — three-tier policy: functional tasks take no image at all
              (AGENTS.md `# Image economy`)

2. BOUND    — what images do get taken are small:
              --viewport-size 1280,720  →  captured pixels capped
              type: jpeg                →  ~5-10x smaller encoding
              media.image.max_base64_bytes: 400000 → read-path cap

3. RECLAIM  — disk:  --output-max-size (automatic, 20 MB)
              DB:    opencode session delete (manual, this document)
```

Net effect: growth becomes **slow and bounded rather than zero**. Expectations
should be set accordingly — there is no automatic DB compaction.

---

## Measurement (how to confirm the design is working)

> ⚠️ **`du` on `opencode.db` is NOT a valid metric.** Verified empirically: the
> file sat at a byte-identical **459 MB** across six probe sessions, including
> one containing a screenshot, and did not shrink after deleting them. SQLite
> reuses free pages inside the already-grown file, so the file size is
> insensitive to both inserts and deletes.

Use the **logical** size of a session instead — that is the number the design
actually controls:

```sh
# find the session
opencode session list

# measure what that session truly costs (text + stored images)
opencode session export <session-id> | wc -c

# measure just the image payload inside it
opencode session export <session-id> | grep -o 'data:image/[a-z]*;base64' | wc -l
```

### Measured results (2026-10-17)

| Session | Export size | Share that is image |
|---------|-------------|---------------------|
| Text-only session (tool list request) | **4,529 B** | 0% |
| Tier 2 session (navigate + one jpeg screenshot) | **103,611 B** | **97,815 B (94.5%)** |

So a **single screenshot costs about 23 text-only sessions**, and ~95% of that
session is pure image bytes. This is precisely why the three-tier policy exists,
and it is the ratio to watch when checking the design is holding.

### The reclaim lever does not shrink the file

`opencode session delete` frees the logical space and stops further growth, but
the **file stays ~459 MB**. That is expected SQLite behaviour, not a bug. The
honest expectations are:

- deletes **prevent growth** and free logical rows
- the file itself only shrinks via `VACUUM`, which the OpenCode CLI does not
  expose

So judge the design by *per-session export size*, not by `du`.

---

## Open Question / Unverified

Whether OpenCode's `tool_output.max_bytes` (schema default `51200`) truncates
MCP **image** content blocks or only text output is unverified. If it does
truncate images, Tier 2 screenshots are additionally hard-capped at the
OpenCode layer, making `media.image` belt-and-braces.

Settled empirically by taking one screenshot and measuring the resulting
session entry size.

---

## Rollback

Nothing to roll back — this change is documentation and an operational habit.
No configuration was modified.
