# Change: Add Playwright MCP Server Configuration

**Date:** 2026-10-17
**File Modified:** `~/.config/opencode/opencode.json` (new file)

---

## Summary

Configured OpenCode to use Playwright MCP for browser automation, enabling the
primary agent to test UI changes like a human user would.

---

## File Content

**Path:** `/Users/parth/.config/opencode/opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "playwright": {
        "type": "local",
        "command": ["npx", "-y", "@playwright/mcp@latest"],
        "codemode": false
      }
    }
  }
}
```

---

## Configuration Details

| Field | Value | Purpose |
|-------|-------|---------|
| `type` | `"local"` | Runs as a local stdio process, not a remote server |
| `command` | `["npx", "-y", "@playwright/mcp@latest"]` | Uses npx to run the latest Playwright MCP package without global installation |
| `codemode` | `false` | Exposes Playwright tools directly on the model's tool list so screenshots and snapshots arrive as native images/text (not through Code Mode sandbox) |

**Why `codemode: false`:**
- Playwright's value for UI testing includes visual verification via screenshots
- Code Mode routes tool calls through an execute sandbox, which may not preserve image results natively
- Direct tool exposure lets the agent see screenshots and accessibility snapshots as the model would receive them

**Why `npx` with `@latest`:**
- No global npm install required
- Automatically gets updates when Playwright MCP releases fixes
- `npx -y` auto-confirms the install prompt for non-interactive execution

---

## Permission Impact

No explicit permission rules were added.

**Base policy (default):**
```json
[
  { "action": "*", "resource": "*", "effect": "allow" },
  { "action": "read", "resource": "*.env", "effect": "ask" },
  { "action": "read", "resource": "*.env.*", "effect": "ask" },
  { "action": "read", "resource": "*.env.example", "effect": "allow" }
]
```

Playwright MCP tools are named `<server>_<tool>` after normalization:
- `playwright_browser_navigate`
- `playwright_browser_snapshot`
- `playwright_browser_click`
- `playwright_browser_type`
- `playwright_browser_fill_form`
- `playwright_browser_take_screenshot`
- `playwright_browser_console_messages`
- etc.

All inherit from the `*: * → allow` rule, so no additional permission configuration is needed.

---

## Verification

```bash
$ opencode mcp list
✓ playwright  connected
```

---

## Prerequisites

- **Node.js ≥ 18** (tested with v26.7.0)
- **npx** (tested with 11.19.0)
- **Playwright browser**: The MCP will download Chromium on first use, or you can pre-install:
  ```bash
  npx playwright install chromium
  ```

**Browser cache status at setup:**
- Existing: `chromium-1243` in `~/Library/Caches/ms-playwright/`
- Required by `@playwright/mcp@0.0.83`: revision `1247` (will download on first run)
- Alternative: Google Chrome installed — can use `--browser chrome` flag to skip Chromium download

---

## Optional Enhancements

Add these to `command` array if needed:

| Flag | Purpose |
|------|---------|
| `--headless` | Run browser without visible window (less intrusive) |
| `--isolated` | In-memory profile, no state persisted between runs |
| `--browser chrome` | Use installed Google Chrome instead of downloading Chromium |

Example with enhancements:
```json
"command": ["npx", "-y", "@playwright/mcp@latest", "--headless", "--isolated"]
```

---

## Rollback

Delete `~/.config/opencode/opencode.json` or remove the `mcp` section, then restart:
```bash
opencode service restart
```
