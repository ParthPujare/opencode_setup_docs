# Verification Evidence

Screenshots the agent could not verify — failed checks, or work waiting on your
approval — are copied here for review.

## Where files come from

Playwright MCP writes every screenshot automatically to its scratch output dir:

```
~/.cache/opencode/playwright-screens/page-<timestamp>.jpeg
```

On a **failed** verification (or when approval is needed), the agent `cp`s that
already-written file here. It does **not** take a second screenshot — the bytes
are already on disk, so preserving them costs nothing.

On a **passing** verification nothing is copied. Scratch files are reclaimed
automatically by Playwright's 20 MB output budget.

## Naming convention

```
<YYYY-MM-DD>-<short-slug>/
  ├── page-<timestamp>.jpeg
  └── ...
```

Example:

```
2026-10-17-checkout-modal-overlap/
  ├── page-1792267200123.jpeg
  └── page-1792267204456.jpeg
```

The slug should identify the failing behaviour, not the file: use
`checkout-modal-overlap`, not `screenshot-1`.

## Retention

Files in this directory are **outside** the Playwright `--output-dir`, so the
`--output-max-size` budget can never evict them. This folder is permanent until
you delete it.

Clean up old evidence whenever you like:

```sh
# list what is here
ls -lt ~/Documents/Work/opencode/evidence/

# remove evidence older than 30 days
find ~/Documents/Work/opencode/evidence/ -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +
```

## Note on agent access

The agent captures Tier 2 screenshots inline (it sees the image in-conversation)
without naming a file. It must **not** `read` files back out of this folder —
that would re-inject the image bytes into session storage, which is exactly what
this whole design avoids. Files here are for you to open.
