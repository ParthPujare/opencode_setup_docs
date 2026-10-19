# Change: Add Verification-First Completion Policy

**Date:** 2026-10-17
**File Created:** `~/.config/opencode/AGENTS.md` (new file)

---

## Summary

Added a global instruction file that enforces verification before completion
claims. The primary agent must execute and observe UI changes using Playwright
MCP before reporting a task as "finished" or "completed".

---

## File Content

**Path:** `/Users/parth/.config/opencode/AGENTS.md`

```markdown
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
5. Take `playwright_browser_take_screenshot` as evidence when the result is
   visual.

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

Close every task with: what you tested, the exact steps/URL used, and the
evidence observed (visible outcome, console status, screenshot).
```

---

## Rationale for AGENTS.md (Not agents.build.system)

### Prompt Assembly Order

OpenCode assembles the system prompt in this order:
1. Agent or provider system prompt
2. Code Mode tool guidance (when enabled)
3. MCP server, reference, and skill guidance
4. Global and project AGENTS.md files, then configured instructions
5. Built-in date and environment context
6. Session-specific instruction entries

**Why not `agents.build.system`:**
- Setting `system` on the build agent **replaces** the entire built-in prompt
- The built-in prompt contains critical tool usage, editing, and safety guidance
- Replacing it would remove OpenCode's core behavior unless we manually reconstruct all of it

**Why AGENTS.md works:**
- Appends **after** the built-in prompt (no replacement)
- Hot-reloads: edits to AGENTS.md update running sessions automatically
- Applies to all agents (primary, subagents) — verification is a universal requirement
- Detected on session start and when files are read

### Scope

**Global location** (`~/.config/opencode/AGENTS.md`) applies to:
- All projects
- All agents (`build`, `plan`, `general`, `explore`, custom agents)
- Both primary and subagent sessions

**Plan mode note:** The plan agent denies file edits by default, so the
verification policy is inert in plan mode (agent can't make changes to verify).
In build mode, it's fully active.

---

## Verification Policy Breakdown

### Core Rule
- **Before:** "finished", "completed", "done", "fixed", "working"
- **Requirement:** Execute the feature yourself, observe it working
- **Insufficient:** Code review, typecheck passing, assuming behavior

### UI-Facing Changes (Playwright MCP Required)

**Definition:** Any change with user-visible surface — page, route, form,
modal, flow, button, component.

**Process:**
1. **Start app like a user**
   - Dev server, or build + preview
   - Run in background, capture real URL

2. **Human-style interaction**
   - `browser_navigate` → goto URL
   - `browser_snapshot` → read accessibility tree
   - `browser_click` → click real elements (buttons, links)
   - `browser_type` or `browser_fill_form` → enter real input
   - Follow complete flow including edge/error paths

3. **Confirm outcome from page state**
   - Visible text matches expected
   - Target element appears/disappears
   - URL changes correctly
   - Data persists (if applicable)
   - **NOT sufficient:** "No exception thrown"

4. **Check console**
   - `browser_console_messages` at error level
   - New errors/warnings caused by the change → **failure**

5. **Visual evidence**
   - `browser_take_screenshot` when result is visual

### Non-UI Changes (Direct Execution Required)

**Definition:** Script, API, config, data, build — no browser surface.

**Process:**
- Execute the artifact directly
- Run the script / call the endpoint / invoke the command / run tests
- Observe actual output
- Same rule: observed behavior, not code reading

### On Failure
- Do NOT report completion
- Diagnose and fix
- Re-run the same human-style test from the start
- Only claim success after it passes
- State what broke and how you confirmed the fix

### When Testing Is Impossible
- Say so explicitly
- Name the blocker (missing credentials, no app, Playwright unavailable)
- Never present untested work as done

### Required Report Format
Close every task with:
- What you tested
- Exact steps / URL used
- Evidence observed (visible outcome, console status, screenshot)

---

## Policy Design Decisions

**Why require report format:**
- Forces the agent to document verification
- Makes skips visible ("I couldn't test because X")
- Creates audit trail of what was actually verified

**Why separate UI vs non-UI:**
- UI: Playwright MCP is the natural tool (browser interaction)
- Non-UI: Browser doesn't apply — must execute directly
- Both require **observation**, not just code review

**Why require edge/error paths:**
- Real users don't follow happy path only
- A flow that works on success but crashes on error is broken
- Human-like testing means testing what humans actually do

**Why console errors are failures:**
- New errors/warnings indicate regression
- Users see broken behavior even if UI appears functional
- Catch issues early before user reports them

---

## Rollback

Delete `~/.config/opencode/AGENTS.md` or empty the file.

**Note:** Editing this file while a session is running triggers an instruction
update on the next prompt — no restart needed.
