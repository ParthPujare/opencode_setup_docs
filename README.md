# OpenCode Configuration Changes

**Date:** 2026-10-17
**Purpose:** Enable verification-first completion policy using Playwright MCP

---

## Overview

Configured OpenCode's primary agent to test UI changes like a human would
using Playwright MCP, and to only report tasks as "finished" or "completed"
after observing the changes work as intended.

---

## Changes Made

### Change 1: Playwright MCP Server Configuration

**File:** `~/.config/opencode/opencode.json` (new)

Configures Playwright MCP as a local server with direct tool exposure
(disabled Code Mode for Playwright) so the agent can drive browser testing.

**Documentation:** `01-playwright-mcp-config.md`

---

### Change 2: Verification-First Completion Policy

**File:** `~/.config/opencode/AGENTS.md` (new)

Global instruction file that enforces:
- Execute and observe UI changes with Playwright MCP before claiming completion
- Human-style testing (navigate, click, type, submit, check console)
- Fallback to direct execution for non-UI changes
- Explicit reporting of what was tested and evidence observed

**Documentation:** `02-agents-md-verification-policy.md`

---

### Change 3: Global Read-Only `verify` Agent (PMAT)

**Files:** `~/.config/opencode/agents/verify.md` (new agent) and
`~/.local/bin/pmat` (new binary, PMAT 3.42.0 prebuilt — no Rust toolchain)

Adds a primary agent that detects the project's stack, runs `pmat verify
--format json --no-fail-fast` plus the project's own tests/lint/typecheck, and
reports a single `VERDICT: PASS/FAIL` line with `file:line` findings. Read-only:
`edit` and `subagent` denied; shell allowed so it can run checks. PMAT CLI only —
no MCP registration. Works in any project directory.

**Documentation:** `03-verify-agent-pmat.md`

---

### Change 4: Playwright Image Bounds & Scratch Output Directory

**File:** `~/.config/opencode/opencode.json` (modified)

Adds `--viewport-size 1280,720`, `--output-dir` (scratch dir outside
projects), `--output-max-size` 20 MB, `--file-paths absolute`, and a
`media.image.max_base64_bytes` cap. Bounds screenshot bytes at capture time
(MCP images are **not** covered by `media.image`), moves scratch files out of
the workspace, and gives the `read`-image path a hard byte ceiling.

**Documentation:** `05-image-economy-and-visual-diagnosis.md`

---

### Change 5: Three-Tier Verification & Image Economy Policy

**File:** `~/.config/opencode/AGENTS.md` (modified)

Replaces unconditional screenshotting with a **classified, three-tier** policy.
The agent classifies each task once (functional / visual / ambiguous → ask),
defaults to text-only verification, must screenshot for visual tasks, and
reports tier + reason + screenshot count on every task.

**Documentation:** `06-three-tier-verification-policy.md`

---

### Change 6: Session Storage Hygiene

**File:** none (operational procedure)

Documents that **compaction does not shrink `opencode.db`** — only
`opencode session delete` reclaims it. Records the three-layer growth strategy
(prevent → bound → reclaim) and the measurement procedure.

**Documentation:** `07-session-storage-hygiene.md`

---

### Change 7: Vision Prerequisite, `scale` Gotcha & Measured Findings

**Files:** `~/.config/opencode/AGENTS.md` (modified), corrections to `05`/`06`/`07`/`README.md`

Live testing exposed that the default `build` model (`opencode/big-pickle`) is
**text-only** — it returned `CANNOT-SEE-IMAGE`, making Tier 2 visual verification
impossible and risking a *false visual pass*. Adds a vision prerequisite to Tier 2,
records that OpenCode requires `scale` on screenshots, and corrects the
`07` measurement method (`du` on `opencode.db` is invalid — SQLite reuses free pages).

**Documentation:** `08-vision-prerequisite-and-measured-findings.md`

---

### Change 8: ASD-STE100 Writing Style for All Agents

**Files Created:** `~/.config/opencode/references/asd-ste100-writing.md`,
`~/.config/opencode/agents/plan.md` (custom override of the builtin `plan`
agent)
**Files Modified:** `~/.config/opencode/agents/{scope,build,verify}.md`

Makes the `plan`, `scope`, `build`, and `verify` agents write their
user-facing prose in ASD-STE100 Simplified Technical English — the aerospace
standard for controlled technical language (53 rules, ~900-word approved
dictionary). Enforces a 20/25-word sentence cap, active voice, imperative
mood, conditions-first ordering, no semicolons/contractions/Latin
abbreviations, an approved-verb table, and consistent terminology.

**Prose-only policy:** STE governs what each agent writes *for the user*
(plans, questions, scope locks, reports, findings, fixes). It does **not**
govern code, code comments, commit messages, variable names, file paths,
command lines, or quoted code snippets. The `build` agent carries the
strongest exemption and a revert-if-leaked double-check — rechecked twice
before delivery.

**Documentation:** `09-asd-ste100-writing-style.md`

---

## Files Created

```
~/.config/opencode/
├── opencode.json          # Playwright MCP config + image bounds + scratch dir
├── AGENTS.md              # Verification policy + 3-tier image economy
├── PIPELINE.md            # Pipeline overview
├── agents/
│   ├── plan.md            # Custom plan agent (overrides builtin) + STE style
│   ├── scope.md           # Scope-lock agent + STE style
│   ├── build.md           # Build agent + STE style (code exempt, rechecked)
│   └── verify.md          # Read-only verify agent (pmat + checks) + STE style
├── commands/
│   ├── plan.md            # /plan command
│   ├── scope.md           # /scope command
│   ├── build.md           # /build command
│   └── verify.md          # /verify command
└── references/
    ├── engineering-principles.md   # 111-item rubric for the scope agent
    └── asd-ste100-writing.md       # Distilled STE rules for agent prose

~/.local/bin/
└── pmat                   # PMAT 3.42.0 binary (prebuilt aarch64-apple-darwin)

~/.cache/opencode/
└── playwright-screens/    # Playwright scratch output (20 MB auto-evicting)

~/Documents/Work/opencode/
├── README.md                              # This file
├── 01-playwright-mcp-config.md
├── 02-agents-md-verification-policy.md
├── 03-verify-agent-pmat.md
├── 05-image-economy-and-visual-diagnosis.md   # (04 unused — reserved for
│                                               #  plan-scope-build-pipeline.md)
├── 06-three-tier-verification-policy.md
├── 07-session-storage-hygiene.md
├── 08-vision-prerequisite-and-measured-findings.md
├── 09-asd-ste100-writing-style.md
├── plan-scope-build-pipeline.md
└── evidence/                              # Failed-verification screenshots
    └── README.md
```

---

## Replication Instructions

> **Scope:** Steps 1–6 below replicate **Change 1, 2, 4, 5, 6 and 7** (config +
> policy + evidence folder). Step 2's `AGENTS.md` heredoc already includes
> Change 7's vision prerequisite and `scale` bullet. Change 3 (the `verify`
> agent + PMAT binary) has its own complete, checksum-verified steps in
> `03-verify-agent-pmat.md` → "Replication Instructions". The `scope` agent and
> `/plan` `/scope` `/build` commands are documented in
> `plan-scope-build-pipeline.md`.
>
> **Model choice is part of replication:** the default `build` model is
> text-only and cannot verify visual work. Use
> `--model opencode/nemotron-3.5-super-vl-free` for Tier 2 tasks — see
> `08-vision-prerequisite-and-measured-findings.md`.

### Step 1: Create opencode.json

> Covers **Change 1** and **Change 4**.

```bash
mkdir -p ~/.config/opencode ~/.cache/opencode/playwright-screens
cat > ~/.config/opencode/opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "playwright": {
        "type": "local",
        "command": [
          "npx",
          "-y",
          "@playwright/mcp@latest",
          "--viewport-size",
          "1280,720",
          "--output-dir",
          "/Users/parth/.cache/opencode/playwright-screens",
          "--output-max-size",
          "20971520",
          "--file-paths",
          "absolute"
        ],
        "codemode": false
      }
    }
  },
  "media": {
    "image": {
      "auto_resize": true,
      "max_base64_bytes": 400000
    }
  }
}
EOF
```

### Step 2: Create AGENTS.md

> Covers **Change 2** and **Change 5**. Read the trailing `'EOF'` — it is
> quoted, so backticks and `$` are literal and need no escaping.

```bash
cat > ~/.config/opencode/AGENTS.md << 'EOF'
# Verify before claiming completion

You may not describe a task as "finished", "completed", "done", "fixed", or
"working" until you have executed the changed feature yourself and observed it
behave as intended. Reading the code, passing typecheck, or assuming behavior
is not verification.

## UI-facing changes — test like a human, with Playwright MCP

Any change with a user-facing surface (page, route, form, modal, flow, button,
component) must be exercised through the Playwright MCP browser tools the way a
real user would:

1. Start the app the way a user would reach it (dev server, or build + preview)
   in the background, and use its real URL.
2. Drive the flow end to end with `playwright_browser_navigate`,
   `playwright_browser_snapshot`, `playwright_browser_click`,
   `playwright_browser_fill_form` / `playwright_browser_type`: navigate, click
   real elements, type real input, submit, follow the flow — including the
   edge and error paths a user would actually hit.
3. Confirm the intended outcome from the page state itself: visible text, the
   element that should appear, the expected URL, the persisted record. "No
   exception was thrown" is not a pass.
4. Run `playwright_browser_console_messages` at error level and treat any new
   error or warning caused by your change as a failure.
5. Only if the task is classified **visual** (see Image economy below): take
   `playwright_browser_take_screenshot`. Functional tasks stop here — no image.

## Non-UI changes — verify by running it

No browser surface (script, API, config, data, build): execute the artifact
directly — run the script, call the endpoint, invoke the command, run the
relevant tests — and observe the actual output. Same rule: observed behavior,
not code reading.

## When the test fails

Do not report completion. Diagnose, fix, re-run the same human-style test from
the start, and only claim success after it passes. State what broke and how you
confirmed the fix.

## When you cannot test

Say so explicitly and name the blocker (missing credentials, no runnable app,
Playwright MCP unavailable, etc.). Never present untested work as done.

## Report format

Close every task with:
- what you tested and the exact steps/URL used
- evidence observed (visible outcome, console status)
- **Verification tier used:** Tier 1 (text) or Tier 2 (visual)
- **Why:** one line
- **Screenshots taken:** count

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
EOF
```

> **Note:** The `# Plan → Scope → Build pipeline` section that follows in the
> live file was added separately — see `plan-scope-build-pipeline.md`. Append it
> after the block above if you want the complete current file.

### Step 3: Create the evidence folder

```bash
mkdir -p ~/Documents/Work/opencode/evidence
# then see evidence/README.md for the naming convention
```

### Step 4: Restart OpenCode Service

```bash
opencode service restart
```

### Step 5: Verify MCP Connection

```bash
opencode mcp list
# Expected: ✓ playwright  connected
```

### Step 6: (Optional) Pre-install Playwright Browser

Skip if you want automatic download on first use, or use `--browser chrome`
flag in opencode.json to use installed Google Chrome.

```bash
npx playwright install chromium
```

---

## Verification

### Test the MCP server responds:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | \
  npx -y @playwright/mcp@latest 2>/dev/null | head -1
```

Expected: JSON response with server capabilities

### Test in OpenCode session:

Start a new OpenCode session and ask:

> "Navigate to https://example.com and take a screenshot"

Expected: Agent uses `playwright_browser_navigate` and
`playwright_browser_take_screenshot` tools

### Test verification policy:

Start a new session and ask:

> "Add a button to any page and tell me when it's done"

Expected: Agent does NOT say "done" until it has:
1. Started the dev server
2. Used Playwright to navigate to the page
3. Verified the button appears and is clickable
4. Checked console for errors
5. Reported what was tested with evidence

---

## Default Settings Modified

| Setting | Default | New Value | Reason |
|---------|---------|-----------|--------|
| `mcp.servers` | `{}` (empty) | `playwright` server added | Enable browser automation for UI testing |
| `codemode` | `true` (global default) | `false` (playwright only) | Expose Playwright tools directly for native screenshot/snapshot results |
| `--viewport-size` | Playwright default (1280×720) | `1280,720` explicit | Bound screenshot bytes **at capture** — `media.image` does not reach MCP results |
| `--output-dir` | `<workspace>/.playwright-mcp/` | `~/.cache/opencode/playwright-screens` | Keep scratch files out of home dir and all projects |
| `--output-max-size` | unset (unlimited) | `20971520` (20 MB) | Auto-evict oldest scratch files. **Disk only — not `opencode.db`** |
| `--file-paths` | `relative` | `absolute` | Report absolute paths so `cp` works from any workspace |
| `media.image.max_base64_bytes` | `5242880` | `400000` | Cap `read`-of-image and attachment payloads |
| `media.image.max_width/height` | `2000` | `2000` (unchanged) | Deliberately not tightened — would degrade user's own pasted diagrams |
| AGENTS.md | None | Verification policy | Enforce testing before completion claims |
| AGENTS.md report format | single sentence | tier / why / screenshot count | Make the tier decision auditable |
| AGENTS.md screenshot rule | unconditional when visual | classified 3-tier policy | Stop images on functional tasks; force them on visual ones |

**Note:** `codemode: false` is set per-server (only on `playwright`), not
globally. Other MCP servers would still use Code Mode by default.

**Known limitation:** the three-tier policy is **advisory** — OpenCode
permission rules match tool *name* only, never arguments, so "screenshot only
for visual tasks" cannot be expressed in config. Bounding comes from
`--viewport-size` + JPEG; compliance comes from the mandatory report line.

---

## Verified Findings (measured 2026-10-17)

> Full detail, exact test commands and rollback for these three findings:
> **`08-vision-prerequisite-and-measured-findings.md`**.

### ⚠️ Tier 2 needs a vision-capable model

Tested headlessly with `opencode run`:

| Model | Asked to describe a screenshot |
|-------|--------------------------------|
| `opencode/big-pickle` (the default `build` model) | **`CANNOT-SEE-IMAGE`** |
| `opencode/nemotron-3.5-super-vl-free` | Described the page correctly |

On a text-only model the screenshot bytes are stored and paid for but **cannot be
perceived**, so visual verification is impossible and the agent would be tempted
to false-pass off the accessibility snapshot. `AGENTS.md` → Tier 2 now carries a
vision prerequisite that forces this to be named as a blocker instead.

Run visual tasks with `--model opencode/nemotron-3.5-super-vl-free` (or any
`-vl-` model), or accept Tier 1 only.

### `scale` is required on screenshots through OpenCode

Direct MCP accepts `{"type":"jpeg"}`; the same call through OpenCode fails with
`Invalid arguments ... - scale: Missing key`. Always pass `scale: "css"`.

### Measured cost of one screenshot

`opencode session export | wc -c`:

| Session | Export size | Image bytes |
|---------|-------------|-------------|
| Text-only | 4,529 B | 0 |
| navigate + one jpeg | **103,611 B** | **97,815 B (94.5%)** |

One screenshot ≈ **23 text-only sessions**. That ratio is the whole justification
for the three-tier policy.

### `du` on `opencode.db` is not a valid metric

The file stayed at a byte-identical **459 MB** across six probe sessions
(including one with a screenshot) and did not shrink after deleting them.
SQLite reuses free pages. Measure per-session export size instead — see
`07-session-storage-hygiene.md`.

---

## Rollback

```bash
# Remove configuration
rm ~/.config/opencode/opencode.json
rm ~/.config/opencode/AGENTS.md

# Restart service
opencode service restart
```

---

## Scope

**Global:** Both files are in `~/.config/opencode/`, so they apply to:
- All projects
- All agents (build, plan, subagents)
- All sessions (new and existing, after AGENTS.md edit detection)

**To scope to a single project:**
- Move `AGENTS.md` to project root
- Move `opencode.json` to project root or `.opencode/`

---

## Troubleshooting

### Playwright MCP not connecting

```bash
# Check service status
opencode service status

# Check logs
tail -50 ~/.local/share/opencode/log/opencode.log | grep -i playwright

# Test MCP server manually
npx -y @playwright/mcp@latest --help
```

### Browser not found

```bash
# Install Chromium
npx playwright install chromium

# OR use installed Chrome (edit opencode.json)
"command": ["npx", "-y", "@playwright/mcp@latest", "--browser", "chrome"]
```

### Policy not taking effect

```bash
# Verify file exists
cat ~/.config/opencode/AGENTS.md

# Restart service
opencode service restart

# Start NEW session (existing sessions pick up changes on next prompt)
```

---

## Related Documentation

- **OpenCode MCP Docs:** https://opencode.ai/v2/docs/mcp-servers
- **OpenCode Instructions:** https://opencode.ai/v2/docs/instructions
- **OpenCode Agents Docs:** https://opencode.ai/v2/docs/agents
- **Playwright MCP:** https://github.com/microsoft/playwright-mcp
- **PMAT (pmat):** https://github.com/paiml/paiml-mcp-agent-toolkit
