# Change: Three-Tier Verification & Image Economy Policy

**Date:** 2026-10-17
**File Modified:** `~/.config/opencode/AGENTS.md`
**Builds on:** `02-agents-md-verification-policy.md`

---

## Summary

Replaced "always screenshot when visual" with a **classified, three-tier**
verification policy. The agent decides *once per task* which tier applies,
must **report** its choice, and never screenshots to confirm something already
proven from text.

---

## What Changed in AGENTS.md

### Change A — UI verification step 5

**Before:**
```markdown
5. Take `playwright_browser_take_screenshot` as evidence when the result is
   visual.
```

**After:**
```markdown
5. Only if the task is classified **visual** (see Image economy below): take
   `playwright_browser_take_screenshot`. Functional tasks stop here — no image.
```

### Change B — Report format (was a single sentence)

**Before:**
```markdown
Close every task with: what you tested, the exact steps/URL used, and the
evidence observed (visible outcome, console status, screenshot).
```

**After:**
```markdown
Close every task with:
- what you tested and the exact steps/URL used
- evidence observed (visible outcome, console status)
- **Verification tier used:** Tier 1 (text) or Tier 2 (visual)
- **Why:** one line
- **Screenshots taken:** count
```

### Change C — new `# Image economy` section (inserted before the pipeline section)

```markdown
# Image economy — how visual verification is decided

Inline images persist in session storage and are the main source of database
bloat. Playwright runs at 1280x720 and MCP screenshots are not size-capped by
OpenCode, so pick the cheapest tier that answers the question.

## Classify the task once, before verifying

- **Functional** — data, state, text, flow, auth, API, config: use Tier 1
  only. Tier 2 is forbidden unless Tier 1 fails or is ambiguous.
- **Visual** — CSS, layout, styling, responsive, animation, theming, a new or
  restyled component: Tier 2 is MANDATORY, not optional. A visual change
  reported without a screenshot is incomplete.
- **Ambiguous** — neither clearly functional nor clearly visual: ask me which
  tier to use before verifying. One short question. Do not guess.

## Tier 1 — functional checks, no image

Verify with `playwright_browser_snapshot` (accessibility tree),
`playwright_browser_console_messages`, `playwright_browser_network_requests`,
and the URL. Assert on what a human would read: "confirmation dialog shows
`Saved`", not "no exception thrown". This is the default channel for every
task.

## Tier 2 — visual diagnosis, one bounded screenshot

**Prerequisite — vision:** you must actually be able to perceive image content.
Some models are text-only. If you cannot describe what is in a screenshot, you
cannot verify visually: say so explicitly and name it as a blocker (or re-run
the task on a vision-capable model). Never report a visual pass from the
accessibility snapshot alone, and never claim to have "seen" pixels you cannot
see.

Take it WITHOUT a `filename` so the image reaches you inline:

- `type: "jpeg"` (roughly 5-10x smaller than png)
- `scale: "css"` — **required**; omitting it fails schema validation
- never `fullPage: true` unless the defect is below the fold; capture the
  failing element with `target` instead where you can
- one or two per verification, re-using what you already captured

Never screenshot to confirm state already visible in the snapshot, to celebrate
a pass, or "just in case".

## Tier 3 — evidence for the user, file only, zero bytes

The auto-named screenshot was already written to the output path shown in the
tool result. When verification FAILS or you need my approval, `cp` that existing
file to:

    ~/Documents/Work/opencode/evidence/<YYYY-MM-DD>-<short-slug>/

Then report the path. Do not take a second screenshot, and do not `read` an
evidence file back — you already saw it inline.

On success leave scratch files alone; the output budget reclaims them.
```

The `# Plan → Scope → Build pipeline` section that follows is untouched.

---

## Design Rationale

### Why the decision belongs to the model

Playwright MCP's own tool descriptions already steer this way:

> `browser_snapshot`: *"this is better than screenshot"*
> `browser_take_screenshot`: *"You can't perform actions based on the
> screenshot, use `browser_snapshot` for actions."*

Whether something is "a visual question" is not knowable from code — it's a
judgment a human tester makes intuitively. So the model makes it.

### Why classify per task instead of per check

An open-ended *"screenshot only if you think it's needed"* rule gets taken
lazily in both directions: over-screenshotting ("just in case") inflates the
DB, under-screenshotting lets a CSS bug pass unexamined.

Classifying once per task makes the tier a **property of the work**:

| Task type | Tier 2 |
|-----------|--------|
| Functional | Forbidden unless Tier 1 fails |
| Visual | **Mandatory** |
| Ambiguous | Ask the user — never guess |

The ambiguous case is a single question rather than a per-call guess.

### Why enforcement is a report line, not config

Permission rules in OpenCode match **tool name only, never arguments**:

```
| <server>_<tool> | `*` for an MCP tool |
```

There is no expressible rule for *"allow screenshot only for visual tasks"* —
only allow/deny the whole tool, and denying it would also block the visual
self-diagnosis requirement.

So the control is **auditable rather than enforced**: the mandatory report line
(`Tier used / Why / Screenshots taken`) makes a lazy call visible for review.
Combined with `--viewport-size`, a wrong decision still costs ~200 KB rather
than ~1.3 MB.

### Why Tier 3 costs zero bytes

From `browser_take_screenshot`:

```js
await response2.addFileResult(resolvedFile, data);     // writes file — always
if (!params2.filename)
  await response2.registerImageResult(data, fileType);  // inline — only if unnamed
```

The file is written **regardless**. So *seeing* an image (Tier 2, unnamed →
inline) and *preserving* it (Tier 3 → `cp`) are two independent actions.
Preservation reuses bytes already on disk instead of capturing again.

### Why Tier 2 has a vision prerequisite

The policy assumes the agent can see pixels. That assumption is **model-dependent
and was false by default here**: `opencode/big-pickle` (the `build` agent's
model) returned `CANNOT-SEE-IMAGE` when asked to describe a screenshot, while
`opencode/nemotron-3.5-super-vl-free` described it correctly.

Without this prerequisite the policy produces its worst failure mode — a
**false visual pass**: the agent takes the screenshot (bytes are stored, cost is
paid), cannot perceive it, and reports success on the strength of the
accessibility snapshot alone. The prerequisite converts that silent failure into
an explicit blocker.

Note the cost asymmetry that makes this worth guarding: Tier 2 spends ~98 KB of
storage to buy a claim the model may be unable to make. On a text-only model
that is pure loss — so the honest move is to drop to Tier 1 and say why.

### Why `read`-back is forbidden in Tier 3

`media.image.max_base64_bytes` caps images passing through the **`read` tool**.
An agent that `read`s an evidence PNG re-injects it into session storage —
defeating the purpose. The cap bounds it (400 KB) but does not prevent the
waste. Policy forbids it outright: the agent already saw the image inline.

---

## Task Classification Cheat Sheet

| Signals | Classification |
|---------|----------------|
| data, state, text, form submit, auth, API, config, migration, permissions | **Functional** |
| CSS, layout, styling, responsive, animation, theming, new/restyled component, spacing, colour | **Visual** |
| mixed, or the user said "make it look right" without specifics | **Ambiguous → ask** |

---

## Rollback

Restore step 5 and the report-format sentence from `02-agents-md-verification-policy.md`,
and delete the `# Image economy` section. No restart needed — AGENTS.md edits
hot-reload into running sessions on the next prompt.
