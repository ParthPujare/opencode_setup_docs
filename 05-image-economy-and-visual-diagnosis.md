# Change: Playwright Image Bounds & Scratch Output Directory

**Date:** 2026-10-17
**File Modified:** `~/.config/opencode/opencode.json`
**Supersedes:** part of `01-playwright-mcp-config.md` (command args added)

---

## Summary

Bounded how large screenshot images can get, moved scratch files out of the
workspace, and added a byte cap on the `read`-image path. Together these stop
Playwright verification from bloating the session database and littering
projects.

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
```

---

## What Each Setting Does

| Setting | Type | Effect |
|---------|------|--------|
| `--viewport-size 1280,720` | **Hard** | Caps browser viewport at launch. Bounds screenshot bytes *at capture time*, before base64 encoding exists. |
| `--output-dir` → `~/.cache/opencode/playwright-screens` | **Hard** | Auto-named screenshots go to a scratch dir outside all projects. |
| `--output-max-size 20971520` (20 MB) | **Hard** | Evicts oldest scratch files once total exceeds 20 MB. **Disk only — never touches `opencode.db`.** |
| `--file-paths absolute` | Convenience | Reported paths are absolute, so `cp` works from any workspace. |
| `codemode: false` | — | Unchanged from `01`. Playwright tools exposed directly so snapshots/screenshots arrive natively. |
| `media.image.auto_resize: true` | **Hard** | Resizes oversized images instead of rejecting them. |
| `media.image.max_base64_bytes: 400000` | **Hard** | Caps base64 payload for images going through the **`read` tool** and prompt attachments. |

**Deliberately not set:** `max_width` / `max_height` — left at OpenCode's
default of `2000`. Lowering them would degrade the user's own pasted diagrams.
Only the byte count is constrained.

---

## The Source-Level Analysis (why these particular levers)

Verified by reading `playwright-core@1.64.0-alpha` `lib/coreBundle.js`.

### 1. Only one tool ever emits an inline image

```js
// packages/playwright-core/src/tools/backend/... — browser_take_screenshot
const data = target ? await target.locator.screenshot(options)
                    : await tab2.page.screenshot(options);
const resolvedFile = await response2.resolveClientOutputFile({ ... });
await response2.addFileResult(resolvedFile, data);     // ← ALWAYS writes file
if (!params2.filename)
  await response2.registerImageResult(data, fileType);  // ← ONLY if no filename
```

There are exactly 3 references to `imageResults` in the bundle, one call site
for `registerImageResult`. So:

| Call | File on disk | Inline image to model |
|------|--------------|----------------------|
| `browser_take_screenshot` **no `filename`** | ✅ `<output-dir>/page-{ts}.jpeg` | ✅ full base64, **uncapped** |
| `browser_take_screenshot` **with `filename`** | ✅ `<workspace>/<file>` | ❌ link only |
| `browser_snapshot` **no `filename`** | ❌ none | ✅ inline text (aria tree) |
| `browser_snapshot` **with `filename`** | ✅ `<workspace>/<file>.yml` | ❌ link only |
| **Action tools** (`browser_navigate`, `_click`, `_type`, `_fill_form`, …) | ✅ `<output-dir>/page-{ts}.yml` **always** | ❌ link only |

### 1b. Action tools always write a snapshot file

Verified empirically: `browser_navigate` **alone** (no screenshot call) produced
`page-2026-10-17T07-25-12-072Z.yml` (1560 bytes) and returned
`- [Snapshot](/Users/parth/.cache/opencode/playwright-screens/page-...yml)`.

From the source:

```js
const snapshotToFile = this._includeSnapshot !== "explicit" || !!this._includeSnapshotFileName;
```

- `browser_snapshot` → `setIncludeFullSnapshot(...)` sets `_includeSnapshot = "explicit"`
  → `snapshotToFile = false` when unnamed → **inline aria text**
- action tools → `setIncludeSnapshot()` sets `_includeSnapshot = "full"`
  → `snapshotToFile = true` → **always writes `.yml`, returns link**

Consequences worth knowing:

- **Disk:** every action call drops a ~1.5 KB `.yml` in the scratch dir. Trivial,
  and covered by the 20 MB budget.
- **DB:** this *reduces* session growth — page state after an action is a link,
  not inline text. The agent `read`s that `.yml` to inspect state, which is
  text-only and costs ~1.5 KB, not an image payload.
- **Not affected** by `--viewport-size`; unrelated to image economy.

### 2. `media.image` does NOT apply to MCP tool results

OpenCode docs, Attachments page:

> "These settings apply both to supported images attached to prompts and to
> images returned by the built-in `read` tool."

MCP tool results are neither. So `max_base64_bytes` **cannot** cap a Playwright
screenshot. Confirmed by inspection of the config schema — `media.image` is a
sibling of `mcp`, not nested under it, with no MCP reference.

**Consequence:** the only way to bound inline screenshot bytes is to bound the
*source* — hence `--viewport-size`. A 1280×720 JPEG at quality 90 is roughly
60–150 KB (~80–200 KB base64) versus 400 KB–1.3 MB for an equivalent PNG.

### 3. `--output-max-size` only scans disk

```js
async _enforceOutputBudget() {
  const maxSize = this._context.config.outputMaxSize;
  if (!maxSize) return;
  const dir = outputDir(this._context.options);   // ← disk output dir ONLY
  let entries = await listFilesRecursive(dir);
  entries.sort((a, b) => a.mtimeMs - b.mtimeMs);  // oldest first
  for (const entry of entries) {
    if (total <= maxSize) break;
    if (this._writtenFiles.has(entry.path)) continue;  // never current call
    await fs.unlink(entry.path);
  }
}
```

Runs after every tool call. It has **zero reach** over `opencode.db`. This is
why a disk budget alone never solved the database-bloat problem.

### 4. Default output directory (before this change)

```js
function outputDir(options) {
  if (options.config.outputDir) return path.resolve(options.config.outputDir);
  const baseName = ".playwright-mcp";
  if (isSystemDirectory(options.cwd) || !isWritable(options.cwd))
    return path.join(os.tmpdir(), baseName);
  return path.join(options.cwd, baseName);   // ← cwd = workspace root
}
```

Workspace was `/Users/parth`, so files defaulted to `~/.playwright-mcp/` — in
the home directory. `--output-dir` now redirects that.

### 5. `checkFile` write restriction

```js
async function checkFile(options, resolvedFilename, flags) {
  ...
  if (!isPathInside(realOutput, realFilename) && !isPathInside(realWorkspace, realFilename))
    throw new Error(`File access denied: ${resolvedFilename} is outside allowed roots...`);
}
```

Writes are allowed inside the **output dir** or the **workspace**. Since
`~/.cache/opencode/playwright-screens` is the configured output dir, writing
there is always permitted regardless of which project is open.

---

## Rejected Alternatives

| Option | Why rejected |
|--------|--------------|
| `--image-responses omit` | Would structurally prevent the agent from ever seeing pixels — defeats the requirement that it self-diagnose visual issues. |
| `--caps vision` | Enables coordinate-based clicking, unrelated to image size. |
| Lowering `media.image.max_width/height` | Would degrade the user's own attached diagrams. Only bytes are constrained. |
| Plugin-based screenshot throttling | Enforces a ceiling, but requires custom code to maintain and still can't judge intent. Rejected in favour of policy + auditable report. |

---

## Verification

```bash
$ opencode mcp list
✓ playwright  connected
```

### Empirical results (measured 2026-10-17, macOS arm64)

Server init with **all flags** — clean handshake, empty stderr:

```
{"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{"listChanged":true}},
 "serverInfo":{"name":"Playwright","version":"1.64.0-alpha-1790635538000"}}}
```

**Tier 2 screenshot** (`type: jpeg`, no filename) on `https://example.com`:

| Metric | Value |
|--------|-------|
| Inline payload | `image/jpeg`, **97,792 bytes base64** |
| File on disk | **73,343 bytes** |
| Location | `/Users/parth/.cache/opencode/playwright-screens/page-2026-10-17T07-22-13-852Z.jpeg` |
| Absolute path in response | ✅ yes (`--file-paths absolute`) |

That's ~95 KB base64 vs the ~400 KB–1.3 MB a PNG would have cost — the
viewport + JPEG combination delivering as designed.

**Tier 1 snapshot** (`browser_snapshot`): `TEXT(1272 chars)`, **no image block**.

**Budget eviction:** seeded 10 × 2.5 MB stale files → **24 MB**, triggered one
tool call → **19 MB**. Oldest evicted first; the current call's file and both
recent artifacts survived. Matches `_enforceOutputBudget()` exactly.

**Tier 3 evidence:** `cp` of the existing scratch `.jpeg` into
`evidence/<slug>/` succeeded; confirmed outside `outputDir`, so the budget
cannot touch it.

**Old default dir:** `~/.playwright-mcp/` — never created.

**`opencode.db`:** 459 MB before → 459 MB after. *Note: these probes called the
MCP server directly rather than through an OpenCode session, so no image
entered session storage — this confirms no incidental growth, not yet the
in-session delta. See below for the real in-session measurement.*

### In-session measurement (through OpenCode, not direct MCP)

Probes run via `opencode run` confirm the image **is** persisted to session
storage. Measured by `opencode session export | wc -c`:

| Session | Export size | Image bytes |
|---------|-------------|-------------|
| Text-only (tool-list request) | 4,529 B | 0 |
| navigate + one jpeg screenshot | **103,611 B** | **97,815 B (94.5%)** |

The 97,815 B inline payload matches the 97,792 B from the direct-MCP probe —
the two paths agree, confirming the bounding behaves identically.

⚠️ `du` on `opencode.db` showed **no change** across these sessions. SQLite
reused free pages in the pre-grown 459 MB file, so file size is insensitive to
inserts *and* deletes. Measure per-session export size instead — see
`07-session-storage-hygiene.md`.

### ⚠️ Finding: Tier 2 requires a vision-capable model

The screenshots are stored correctly, but **whether the agent can see them
depends on the model**. Tested headlessly with `opencode run`:

| Model | Asked to describe the screenshot |
|-------|----------------------------------|
| `opencode/big-pickle` (default `build`) | **`CANNOT-SEE-IMAGE`** — text-only |
| `opencode/nemotron-3.5-super-vl-free` | Described the page content correctly |

`big-pickle` is text-only, so on the default agent model **visual
self-diagnosis is impossible** — the pixels arrive in context but cannot be
perceived. This is a model capability limit, not a config problem, and no
Playwright flag can work around it.

Consequences:
- Tier 2 must be run on a vision-capable model (`-vl-` models), or
- the task must fall back to Tier 1 and the missing vision named as a blocker

Encoded in `AGENTS.md` → Tier 2 → "Prerequisite — vision".

### ⚠️ Finding: `scale` is required by OpenCode's schema

A direct MCP call accepts `{"type":"jpeg"}`, but the same call routed through
OpenCode fails:

```
Error: Invalid arguments for tool "playwright_browser_take_screenshot":
- scale: Missing key
```

OpenCode's exposed schema treats `scale` as required even though the MCP server
does not. Always pass `scale: "css"`.

Expected disk behaviour:
- `~/.playwright-mcp/` never created
- `~/.cache/opencode/playwright-screens/` receives `page-*.jpeg` and `page-*.yml`
- Scratch total never exceeds 20 MB

---

## Rollback

Remove the added args and the `media` block:

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

Then `opencode service restart`.
