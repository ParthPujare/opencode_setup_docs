# Change: Vision Prerequisite, `scale` Gotcha & Measured Findings

**Date:** 2026-10-17
**Files Modified:** `~/.config/opencode/AGENTS.md` (Tier 2 section), plus
corrections to `05`, `06`, `07` and `README.md`
**Follows:** `05-image-economy-and-visual-diagnosis.md`, `06-three-tier-verification-policy.md`,
`07-session-storage-hygiene.md`

---

## Summary

Live testing exposed three things the earlier design assumed rather than proved.
Two of them were outright errors in my own documentation:

1. **The default agent model is text-only**, so Tier 2 visual verification is
   impossible on it — and would produce a *false visual pass*.
2. **OpenCode requires `scale`** on `playwright_browser_take_screenshot`, though
   the raw MCP server does not.
3. **`du` on `opencode.db` is not a valid metric** — SQLite reuses free pages, so
   the file size is insensitive to both inserts and deletes. `07` had recommended
   exactly this measurement.

---

## Finding 1: Tier 2 requires a vision-capable model

### The problem

The whole three-tier policy assumes the agent can *see* pixels. That assumption
is model-dependent, and it was **false by default**.

### How it was tested

Headlessly, through a real OpenCode session (`opencode run`), so the MCP config
and tool exposure were exactly as a user would get them:

```sh
opencode run --auto --title "IMAGE-VISIBILITY-TEST" \
"Run exactly these calls, then answer:
1. playwright_browser_navigate with url https://example.com
2. playwright_browser_take_screenshot with type jpeg
Now, looking ONLY at the image data in your context: describe in one sentence
what is visibly on the page. If you truly cannot perceive any image, reply
exactly: CANNOT-SEE-IMAGE."
```

### Results

| Model | Result |
|-------|--------|
| `opencode/big-pickle` (the `build` agent's default) | **`CANNOT-SEE-IMAGE`** |
| `opencode/nemotron-3.5-super-vl-free` | Described the page: icon, multilingual paragraphs |

The second model was invoked with:

```sh
opencode run --auto --model opencode/nemotron-3.5-super-vl-free ...
```

### Why this is the worst failure mode

The screenshot **is** stored (~98 KB, measured) and the cost **is** paid — but a
text-only model cannot perceive it. The dangerous outcome is not an error; it is
a **false visual pass**: the agent takes the screenshot, fails to see it, and
reports success on the strength of the accessibility snapshot alone.

Note the cost asymmetry: Tier 2 spends ~98 KB of storage to buy a claim the model
may be unable to make. On a text-only model that is pure loss.

### What changed

`~/.config/opencode/AGENTS.md` → Tier 2 now opens with a prerequisite:

```markdown
**Prerequisite — vision:** you must actually be able to perceive image content.
Some models are text-only. If you cannot describe what is in a screenshot, you
cannot verify visually: say so explicitly and name it as a blocker (or re-run
the task on a vision-capable model). Never report a visual pass from the
accessibility snapshot alone, and never claim to have "seen" pixels you cannot
see.
```

This converts a **silent** failure into an **explicit** one.

### Operational rule

Run visual tasks with a vision-capable model:

```sh
opencode --model opencode/nemotron-3.5-super-vl-free
# or inside a session: /model opencode/nemotron-3.5-super-vl-free
```

Any `-vl-` model should work; only the two above were verified. If no vision
model is available, fall back to Tier 1 and name missing vision as the blocker.

---

## Finding 2: `scale` is required through OpenCode

### The problem

A screenshot call that succeeds against the MCP server directly **fails** when
routed through OpenCode:

```
Error: Invalid arguments for tool "playwright_browser_take_screenshot":
- scale: Missing key

Arguments provided:
{
  "type": "jpeg"
}
```

OpenCode's exposed schema treats `scale` as required even though the MCP server
does not. This cost a wasted tool call during testing.

### What changed

Always pass `scale`. Recorded in `AGENTS.md` → Tier 2, `05`, and `README.md`:

```markdown
- `scale: "css"` — **required**; omitting it fails schema validation
```

### Replicate the check

```sh
# succeeds against raw MCP
{"jsonrpc":"2.0","id":4,"method":"tools/call",
 "params":{"name":"browser_take_screenshot","arguments":{"type":"jpeg"}}}

# same args via OpenCode -> "scale: Missing key"
```

---

## Finding 3: `du` on `opencode.db` is not a valid metric

### The problem

`07-session-storage-hygiene.md` originally recommended measuring design success
with `du -h ~/.local/share/opencode/opencode.db`. That is wrong.

Observed: the file sat at a **byte-identical 459 MB** across six probe sessions —
including one containing a screenshot — and did **not** shrink after deleting
them. SQLite reuses free pages inside the already-grown file, so the file size is
insensitive to both inserts and deletes.

### The correct metric: logical session size

```sh
opencode session list
opencode session export <session-id> | wc -c
```

### Measured results

| Session | Export size | Share that is image |
|---------|-------------|---------------------|
| Text-only (tool-list request) | **4,529 B** | 0% |
| navigate + one jpeg screenshot | **103,611 B** | **97,815 B (94.5%)** |

So **one screenshot ≈ 23 text-only sessions**, and ~95% of that session is pure
image bytes. This is the ratio that justifies the entire three-tier policy.

The 97,815 B in-session figure agrees with the 97,792 B measured from a direct
MCP call in `05` — the two paths behave identically, confirming the bounding
holds end to end.

### Consequence for the reclaim lever

`opencode session delete` frees logical space and halts growth, but **the file
stays ~459 MB**. That is expected SQLite behaviour, not a bug:

- deletes **prevent growth** and free logical rows
- the file only shrinks via `VACUUM`, which the OpenCode CLI does not expose

### What changed in `07`

The Measurement section was rewritten with the warning, the correct command, and
the measured table. The reclaim section now states the file will not shrink.

---

## Verification

```sh
$ opencode mcp list
✓ playwright  connected

$ python3 -m json.tool ~/.config/opencode/opencode.json > /dev/null && echo OK
OK
```

Probe hygiene: six throwaway sessions were created
(`TIER2-DB-DELTA-PROBE`, `TOOL-PROBE`, `TOOL-PROBE-2`, `TIER2-SCREENSHOT-DELTA`,
`IMAGE-VISIBILITY-TEST`, `VISION-VL-TEST`) and **all were deleted** afterward via
`opencode session delete`. Scratch screenshots under
`~/.cache/opencode/playwright-screens/` were also cleared. Only the pre-existing
session remains.

---

## Rollback

Remove the two added blocks from `~/.config/opencode/AGENTS.md` → Tier 2 (the
`**Prerequisite — vision:**` paragraph and the `scale: "css"` bullet). No restart
needed — `AGENTS.md` hot-reloads on the next prompt.

The findings themselves cannot be rolled back; they are properties of the models
and of SQLite.
