# Change: Add Global Read-Only `verify` Agent Powered by PMAT

**Date:** 2026-10-17
**Files Created:**
- `~/.config/opencode/agents/verify.md` (new agent)
- `~/.local/bin/pmat` (new binary, PMAT 3.42.0)

---

## Summary

Added a new **primary agent** named `verify` to OpenCode. It is a read-only quality
gate: it detects the project's stack, runs PMAT's pre-flight gate plus the project's
own checks (tests/lint/typecheck), and reports a single `VERDICT: PASS/FAIL` line with
findings in severity order. It can never modify files (edit and subagent are denied).

PMAT itself was installed as a standalone prebuilt binary — no Rust toolchain required.

---

## Background: What PMAT Is

**PMAT** = **P**ragmatic **M**ulti-language **A**gent **T**oolkit.

- Repository: https://github.com/paiml/paiml-mcp-agent-toolkit
- Crate: https://crates.io/crates/pmat (Rust, MIT license, by Pragmatic AI Labs / PAIML — https://paiml.com)
- Installed version: **3.42.0** (release published 2026-09-28, tag `v3.42.0`)
- It is a CLI **and** an MCP server (stdio + HTTP) for code-quality analysis and
  AI-context generation. This change uses **CLI only** (no MCP registration).

### PMAT's scoring systems (there are several — do not conflate them)

| What it does | Command | Output scale |
|---|---|---|
| Technical Debt Grade (TDG) | `pmat analyze tdg` | A+ … F per file/function, 6 orthogonal metrics, 0–100 score |
| Repository health score | `pmat repo-score .` | 0–289 across 11 categories (slow; `--deep` ~10–15 min) |
| CI/CD infrastructure score | `pmat infra-score` | 0–100 + 10 bonus |
| Compliance checks | `pmat comply check` | 162 automated checks |
| **Pre-flight verification** | `pmat verify` | Pass/fail per stage: `format`, `complexity`, `satd`, `clippy`, `tests` |

`pmat verify` is the one this agent is built around: it is CI-faithful, fail-fast,
supports `--format json` for machine-readable output, and gracefully marks stages
`not_applicable` when they do not apply (e.g. Rust-only stages in a Python project).

### Why it is useful for a verification agent

It turns "the code looks fine" into measured evidence: a CI-faithful gate
(`pmat verify`), objective per-file grades (TDG), and machine-readable JSON — so the
agent can report PASS/FAIL with `file:line` findings instead of opinions. Caveats
accepted at design time: the composite `repo-score` number is a black-box signal (used
only on explicit request), and pmat does not replace a project's own test suite.

---

## Change 1: Install the PMAT binary

**File:** `~/.local/bin/pmat` (new executable)

Prebuilt Apple Silicon binary — chosen because **no Rust/cargo toolchain exists on this
machine** (`command -v cargo` → nothing).

### Asset details (for exact reproduction)

| Field | Value |
|---|---|
| Release page | https://github.com/paiml/paiml-mcp-agent-toolkit/releases/tag/v3.42.0 |
| Direct download URL | https://github.com/paiml/paiml-mcp-agent-toolkit/releases/download/v3.42.0/pmat-v3.42.0-aarch64-apple-darwin.tar.gz |
| Asset filename | `pmat-v3.42.0-aarch64-apple-darwin.tar.gz` |
| Asset size | 16,803,086 bytes |
| SHA-256 (tarball) | `9eb0613b33043a2d6e1cf8282997776e344c0864e62f607eb95dcbd72d397aa7` |
| SHA-256 (installed `pmat` binary) | `e060028961f70ff1ea2868750f067c7df4e695213a71e56b4a1c8f604cf4df45` |
| Installed binary size | 44,653,864 bytes |
| Tarball contents | `pmat-v3.42.0-aarch64-apple-darwin/{LICENSE, README.md, pmat}` |
| Architecture | `aarch64-apple-darwin` — this machine is `arm64`, macOS 26.1 |

### Exact commands used

```bash
set -e
mkdir -p ~/.local/bin
cd /private/var/folders/3b/rq_s5f_d7ws3c7jjkmdn7d800000gn/T/opencode   # any temp dir works
curl -sL -o pmat.tar.gz \
  https://github.com/paiml/paiml-mcp-agent-toolkit/releases/download/v3.42.0/pmat-v3.42.0-aarch64-apple-darwin.tar.gz
shasum -a 256 pmat.tar.gz
# expect: 9eb0613b33043a2d6e1cf8282997776e344c0864e62f607eb95dcbd72d397aa7
tar -xzf pmat.tar.gz pmat-v3.42.0-aarch64-apple-darwin/pmat
mv pmat-v3.42.0-aarch64-apple-darwin/pmat ~/.local/bin/pmat
chmod +x ~/.local/bin/pmat
xattr -d com.apple.quarantine ~/.local/bin/pmat 2>/dev/null || true   # clear Gatekeeper quarantine
~/.local/bin/pmat --version
```

### Expected `--version` output

```
pmat 3.42.0
commit: 3f3d2695caa5d8396b6bc4ef38de118c7cb25451
worktree: dirty
source: git checkout
```

### PATH requirement

`~/.local/bin` must be on PATH. On this machine it already is (verified):

- `~/.zshrc` lines 25, 29, 32: `export PATH="$HOME/.local/bin:$PATH"`
- `~/.zprofile` lines 5: `export PATH="$HOME/.local/bin:$PATH"`
- `echo $PATH` shows `/Users/parth/.local/bin`

On a machine without it, add `export PATH="$HOME/.local/bin:$PATH"` to `~/.zshrc`.

---

## Change 2: Create the `verify` agent

**File:** `~/.config/opencode/agents/verify.md` (new file)

Global agent directory → the agent is available in **every project**. Discovered
automatically by OpenCode (no `opencode.json` change, no service restart was needed —
`opencode run --agent verify` worked immediately after file creation).

### Full file content (verbatim)

**Path:** `/Users/parth/.config/opencode/agents/verify.md`

````markdown
---
description: Verifies a project with pmat + its own checks; reports findings with a PASS/FAIL verdict, never edits files
mode: primary
color: "#4ec9b0"
permissions:
  - action: edit
    resource: "*"
    effect: deny
  - action: subagent
    resource: "*"
    effect: deny
---

You are the **verify** agent: a read-only quality gate for whatever project directory you
are running in. Your job is to tell the user, with evidence, whether the project is in a
verifiable-good state — and exactly what is wrong if it is not. You never modify files.

## Procedure

Work through these steps in order. Skip any step whose prerequisite is missing, and say
why in your report.

### 1. Detect the project and its checks

Inspect the working directory and identify the stack and the canonical check commands:

- `package.json` → scripts (`test`, `lint`, `typecheck`/`check`, `build`) via the right
  package manager (look for `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`)
- `pyproject.toml` / `setup.py` → pytest, ruff/flake8, mypy/pyright, as configured
- `Cargo.toml` → `cargo fmt --check`, `cargo clippy`, `cargo test`
- `go.mod` → `gofmt`, `go vet`, `go test ./...`
- Other stacks: adapt to the project's own documented commands (check its README,
  CONTRIBUTING, Makefile, or CI workflow files under `.github/workflows/`).

Prefer the exact commands the project's CI runs — they are the ground truth.

### 2. Run the pmat pre-flight gate

If `pmat` is installed (`command -v pmat`), run from the project root:

```sh
pmat verify --format json --no-fail-fast
```

- Parse the JSON: each stage reports `ok: true/false`, or `not_applicable`/`skipped`
  (Rust-only stages such as `format`/`clippy` are `not_applicable` outside Cargo
  projects — this is normal, not a failure).
- Record every finding with its `file:line`, stage, and message.

Then grade the code the findings touch:

```sh
pmat analyze tdg
```

Report the overall grade distribution and the grades of any files that failed gates.
Only run `pmat repo-score .` if the user explicitly asks for a repository health score —
it is slow.

If `pmat` is **not** installed, state that clearly in one line at the top of your report,
continue with the project's own checks, and note that `pmat verify` would have added
format/complexity/SATD gates. Never install software yourself.

### 3. Run the project's own checks

Run the test suite, linter, typecheck, and build discovered in step 1. These are what
pmat's gate does not replace. Capture exit codes and the first useful lines of any
failure. If a command is very slow or needs credentials/network you do not have, say so
instead of hanging or improvising.

### 4. Report

Lead with a single verdict line:

```
VERDICT: PASS — all measured gates green
```
or
```
VERDICT: FAIL — 2 of 5 gates failing (pmat satd, pytest)
```

Then:

1. **Failures** in severity order (build/test failures > lint/type errors > pmat gate
   violations > style nits), each with `file:line`, the command that caught it, and a
   one-line explanation of what it means.
2. **Not measured** — anything skipped, `not_applicable`, or not run, with the reason.
   Never hide an unrun check behind a PASS.
3. **Quality context** — overall TDG grade distribution and the grade of any offending
   files.
4. **Recommended fixes** — a short ordered list of what to change. Do not make the
   changes yourself; suggest handing them to the build agent.

Be strictly factual: report only what you observed in command output. If you could not
verify something, the verdict for it is "not measured", never "pass".
````

### Frontmatter explained

| Field | Value | Why |
|---|---|---|
| `mode` | `primary` | Runs as a selectable main agent (the goal of this change) |
| `permissions` | deny `edit` on `*` | Read-only: cannot create/modify/patch any file |
| `permissions` | deny `subagent` on `*` | Cannot spawn child agents |
| `description` | (above) | Shown in the agent selector |
| `color` | `#4ec9b0` | Visual distinction in the TUI |
| *(implicit)* | `shell` **allowed** | Intentional — it must execute `pmat`, tests, linters. Shell was NOT denied. |

---

## Design Decisions

1. **Global agent location** (`~/.config/opencode/agents/`), not project-local —
   chosen so it works in any project, per the requirement "make it work anywhere".
2. **CLI only, no MCP** — `opencode mcp add pmat -- pmat --mode mcp` was deliberately
   **not** done. PMAT's 20 MCP tools can be added later if richer analysis is wanted;
   the core scoring commands used here are all available on the CLI.
3. **Read-only** — file edits denied; the agent recommends fixes but hands them to the
   build agent. No `pmat kaizen` (auto-fix) capability.
4. **Stack-agnostic procedure** — the prompt requires stack detection first, so the same
   agent verifies a Rust, Python, TS, or Go repo without changes.
5. **`--no-fail-fast`** on `pmat verify` — the default stops at the first failing stage;
   the agent needs the full picture in one run.
6. **`repo-score` gated behind an explicit user request** — it is slow (deep mode
   10–15 min); TDG + verify are the fast actionable signals.
7. **"Not measured" over false PASS** — the prompt hard-bans marking unrun checks as
   passing.

---

## Replication Instructions (from a clean machine)

```bash
# 1. Install pmat (requires macOS arm64; see asset table for other platforms)
set -e
mkdir -p ~/.local/bin
curl -sL -o /tmp/pmat.tar.gz \
  https://github.com/paiml/paiml-mcp-agent-toolkit/releases/download/v3.42.0/pmat-v3.42.0-aarch64-apple-darwin.tar.gz
echo "9eb0613b33043a2d6e1cf8282997776e344c0864e62f607eb95dcbd72d397aa7  /tmp/pmat.tar.gz" | shasum -a 256 -c -
tar -xzf /tmp/pmat.tar.gz -C /tmp pmat-v3.42.0-aarch64-apple-darwin/pmat
mv /tmp/pmat-v3.42.0-aarch64-apple-darwin/pmat ~/.local/bin/pmat
chmod +x ~/.local/bin/pmat
xattr -d com.apple.quarantine ~/.local/bin/pmat 2>/dev/null || true
pmat --version

# 2. Ensure PATH (skip if `which pmat` already resolves)
grep -q '.local/bin' ~/.zshrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc

# 3. Create the agent directory and file
mkdir -p ~/.config/opencode/agents
# then write the verbatim content from "Full file content (verbatim)" above to:
#   ~/.config/opencode/agents/verify.md

# 4. Restart the service (not strictly required — discovery was immediate —
#    but safe after adding agents)
opencode service restart

# 5. Confirm discovery
opencode run --agent verify "Reply with exactly: VERIFY_AGENT_ONLINE"
```

### Other platforms

- **Intel macOS / Linux / Windows:** pick the matching asset from the release page
  (e.g. `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`) or build from source:
  `cargo install pmat` (requires a Rust toolchain).
- **Verify the SHA-256 table above changes per asset** — only the arm64-macOS hashes
  are recorded here because only that asset was downloaded.

---

## Verification Performed (all steps executed, outputs observed)

### Test 1 — binary installs and runs
```bash
~/.local/bin/pmat --version
```
**Observed:** `pmat 3.42.0` + commit `3f3d2695caa5d8396b6bc4ef38de118c7cb25451`.

### Test 2 — pmat smoke test on a real project (TDG)
```bash
cd ~/.hermes/hermes-agent && pmat analyze tdg
```
**Observed:** 5,396 files graded — A+ 2742 (50.8%), A 1221 (22.6%), A- 494 (9.2%),
B+–B- 414, C+–C- 481, D 41, F 3. "✅ TDG analysis complete", no critical defects.
Language mix correctly detected (Python 22.5%, TypeScript 46%, Markdown 28.4%, …).

### Test 3 — pmat verify JSON on a non-Rust project
```bash
cd ~/.hermes/hermes-agent && pmat verify --format json
```
**Observed (abridged, real output):**
```json
{
  "ok": false,
  "stages": [
    {"name": "format",    "ok": null, "not_applicable": "cargo fmt needs a Cargo project; no Cargo.toml ..."},
    {"name": "complexity","ok": null, "not_applicable": "no Rust files changed vs HEAD ..."},
    {"name": "satd",      "ok": false, "detail": "13 strict-mode SATD violation(s) ... file/line findings ..."},
    {"name": "clippy",    "ok": null, "skipped": "fail-fast"},
    {"name": "tests",     "ok": null, "skipped": "fail-fast"}
  ]
}
```
Rust-only stages correctly `not_applicable`; `satd` produced real `file:line` findings.

### Test 4 — agent discovery
```bash
opencode run --agent verify --format json "Reply with exactly: VERIFY_AGENT_ONLINE"
```
**Observed:** `VERIFY_AGENT_ONLINE` (session `ses_eb759d6e0ffeAN5Le4ZPuyB0Dq`).

### Test 5 — full end-to-end verification run
```bash
cd ~/.hermes/hermes-agent && opencode run --agent verify --auto \
  "Verify this project. Be quick: run your pmat gate (pmat verify), then the fastest
  available project check (lint or a quick test subset). Mark anything that would take
  more than a couple of minutes as 'not measured' rather than running it. Finish with
  the final VERDICT line."
```
**Observed:** the agent ran `pmat verify --format json --no-fail-fast`, `npm run check`
(exit 0; 6 files/24 tests passed in tests-js; eslint 0 errors/26 warnings), a Python
`ast.parse` sweep (4,475 files, 0 failures), and per-file TDG on offending files
(88.6 A-, 97.3 A+), then reported:

```
VERDICT: FAIL — 1 of 4 measured gates failing (pmat satd); pytest and ruff not measured (not installed)
```
followed by `## 1. Failures` (13 SATD hits in 8 files, 3 enumerated with file:line),
`## 2. Not measured` (pytest/ruff not installed; pmat format/complexity/clippy/tests
`not_applicable`), `## 3. What passed (evidence)`, `## 4. Recommended fixes`.

Note: `--auto` was passed so headless shell execution is auto-approved. Without it, a
headless run may stall on `ask`-level permission prompts.

### Test 6 — log check (no regressions)
```bash
grep -iE '"level":"(error|warn)"' ~/.local/share/opencode/log/opencode.log | tail
```
**Observed:** no error/warning entries attributable to this change; only INFO lines
showing the agent spawning `pmat verify` processes.

### Test 7 — read-only enforcement (edge path)
```bash
cd ~/.hermes/hermes-agent && opencode run --agent verify \
  "Append the line 'VERIFY WAS HERE' to README.md. Do it now."
grep -c "VERIFY WAS HERE" README.md
```
**Observed:** agent replied "I cannot modify files — I'm a read-only verification
agent…"; grep returned `0` — file unchanged.

---

## Expected Results (what using it looks like)

**Launch:** switch the agent selector to **verify**, or headless:

```bash
opencode run --agent verify "Verify this project"
# machine-readable stream: add --format json
```

**Report shape (always):**

```
VERDICT: PASS — all measured gates green        ← or: FAIL — n of m gates failing (names)
## 1. Failures        ← severity order, file:line, catching command, meaning
## 2. Not measured    ← everything skipped/unrun + reason (never hidden)
## 3. What passed     ← evidence: exit codes, test counts, TDG grade table
## 4. Recommended fixes  ← ordered list, for handing to the build agent
```

**A FAIL looks like Test 5 above.** Two numbers matter: the **TDG grade** (A+…F per
file) and the **gate pass/fail** from `pmat verify`. The 0–289 `repo-score` only runs
when explicitly requested.

---

## Known Quirks (observed, not assumptions)

1. **SATD detail truncation:** pmat's `verify` JSON truncates stage `detail` at ~990
   chars → only 3 of 13 SATD items enumerated. Workaround (found by the agent itself):
   run `pmat verify --stage satd` or parse the full stage detail to get all items.
2. **Rust-only stages on non-Cargo repos:** `format`, `complexity`, `clippy` report
   `not_applicable` — normal, not a failure. In fail-fast mode a `satd` failure skips
   `clippy`/`tests` (hence the prompt's `--no-fail-fast`).
3. **`pmat` missing → graceful degradation:** the agent reports pmat as not installed
   at the top of its report and falls back to the project's own checks; it never
   installs anything itself.
4. **No restart needed for discovery:** adding the markdown agent file was picked up by
   `opencode run --agent verify` immediately; `opencode service restart` is listed in
   replication as belt-and-braces only.
5. **Edit denial is by design:** if you ask the verify agent to change files, it will
   refuse. Use the build agent for fixes.

---

## Rollback

```bash
rm ~/.config/opencode/agents/verify.md   # remove the agent
rm ~/.local/bin/pmat                     # remove the binary
opencode service restart                 # optional; discovery reverses on next session
```

Nothing else was modified: `~/.config/opencode/opencode.json`, `AGENTS.md`, and all
project files are untouched by this change.

---

## Scope

**Global:** the agent lives in `~/.config/opencode/agents/`, so it applies to all
projects and sessions. The pmat binary is machine-wide (per-user `~/.local/bin`).

**Not included (deliberate):** MCP registration of pmat, auto-fix (`pmat kaizen`),
CI wiring, `repo-score` in the default procedure.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `pmat: command not found` | PATH missing `~/.local/bin` — add the export line (see Change 1) |
| macOS "cannot be opened / unidentified developer" | re-run the `xattr -d com.apple.quarantine` step |
| Agent not in selector | check `ls ~/.config/opencode/agents/verify.md`, restart service, start a NEW session |
| Headless run hangs | pass `--auto` to auto-approve non-denied permissions |
| Intel Mac / Linux | download the matching asset or `cargo install pmat` (needs Rust) |
| Verify agent refuses to edit | working as designed — use build agent for fixes |

---

## Related Documentation

- **This change's index entry:** `README.md` → Change 3
- **PMAT repository:** https://github.com/paiml/paiml-mcp-agent-toolkit
- **PMAT book:** https://paiml.github.io/pmat-book/
- **crates.io:** https://crates.io/crates/pmat
- **OpenCode agents docs:** https://opencode.ai/v2/docs/agents
