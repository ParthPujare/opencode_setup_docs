# Plan → Scope → Build pipeline (OpenCode global setup)

Created: 2026-10-17
Effect: global (all projects). Three-stage workflow: `/plan` drafts,
`/scope` grills until scope is locked, `/build` implements (soft-gated).

## What was changed (in order)

### 1. Installed two skills (global)

The "grill-me" skill is a thin wrapper; its companion "grilling" skill holds
the actual method, so both are needed.

```sh
npx skills add mattpocock/skills@grill-me -g -y
npx skills add mattpocock/skills@grilling -g -y
```

Resulting files (OpenCode discovers `~/.agents/skills` as a global source):

```
~/.agents/skills/grill-me/SKILL.md    # wrapper: "Call the Skill tool with grilling."
~/.agents/skills/grilling/SKILL.md    # the real interviewing method
```

Notes:
- `grill-me` has `disable-model-invocation: true` → not advertised to the
  model, loadable only by explicit ID. `grilling` IS advertised and is the
  one agents load.
- The CLI prints a non-fatal "PromptScript does not support global skill
  installation" failure — ignore it; the files are copied.

### 2. New primary agent: `~/.config/opencode/agents/scope.md`

```md
---
description: Interviews the user to lock implementation scope before build
mode: primary
permissions:
  - action: edit
    resource: "*"
    effect: deny
  - action: edit
    resource: "**/plan*.md"
    effect: allow
  - action: edit
    resource: "~/.opencode/plan/**"
    effect: allow
  - action: shell
    resource: "*"
    effect: deny
  - action: subagent
    resource: "*"
    effect: deny
  - action: subagent
    resource: "explore"
    effect: allow
  - action: subagent
    resource: "general"
    effect: allow
---

You are a senior engineer responsible for locking implementation scope
before any code is written. You never write the code yourself.

## Workflow

1. Find the current plan draft. Check `~/.opencode/plan/` and any plan file
   the user points to. If no draft exists, ask the user to run `/plan` first.
2. Load the `grilling` skill with the skill tool and follow its method
   exactly: work the design tree in rounds, ask the whole frontier in one
   numbered round, give your recommended answer for each question, look up
   facts yourself via subagents, and only ask the user for decisions.
3. Make sure these engineering-scope dimensions are covered before locking —
   ask, never assume:
   - Priority ordering — what must ship first, what can slip
   - CAP-style tradeoffs — where is stale/partial data acceptable, what must
     be strongly consistent, failure modes under partition
   - Constraints — perf/latency budgets, scale, security, compatibility
   - Explicit non-goals — deliberately out of scope
   - Failure modes & edge cases — acceptable degradation
   - Acceptance criteria — observable evidence the work is done

## Rules

- Ask before deciding. Every unstated assumption becomes a question.
- Read-only except plan files (permissions enforce it). No shell — dispatch
  an `explore` subagent for environment facts.
- Never finalize until the user declares scope locked (or the grilling
  frontier is empty AND the user confirms shared understanding).
- On lock, append `## Scope Lock` to the plan file, first line
  `Status: locked`, with: decisions, priority order, non-goals,
  failure-mode decisions, acceptance criteria.
- Rounds of ~3–6 numbered questions, each with a recommended answer.
```

Design rationale:
- Edit-deny-everything + allow-only-plan-files means the scope agent can
  question but never quietly "fix" the plan by touching code (rules are
  ordered; last match wins, so the allows come after the deny).
- Shell denied, subagent limited to `explore`/`general`: it investigates via
  read-only subagents instead of acting itself.
- No `model:` set → inherits the session's model. Pin a reasoning-heavy
  variant later if questioning quality is weak.

### 3. Three global commands

`~/.config/opencode/commands/plan.md`:

```md
---
description: Draft an implementation plan, then hand off to /scope
agent: plan
---

Create a draft implementation plan for: $ARGUMENTS

Explore the codebase as needed and write the draft plan to the plan file
(use ~/.opencode/plan/ when no location is specified). Do not treat the
draft as final — the scope stage will interview the user next.

When finished, end with exactly this handoff line:
"Draft ready. Run /scope to lock the scope before building."
```

`~/.config/opencode/commands/scope.md`:

```md
---
description: Grill the user until implementation scope is locked
agent: scope
---

Load the @grilling skill and interview me relentlessly about: $ARGUMENTS

Follow your instructions: cover priority ordering, CAP-style tradeoffs,
constraints, non-goals, failure modes, and acceptance criteria. Ask in
numbered rounds with your recommended answer for each question. Do not
finalize anything until I confirm the scope is locked — then write the
`## Scope Lock` section (first line `Status: locked`) into the plan file
and tell me to run /build.
```

`~/.config/opencode/commands/build.md`:

```md
---
description: Implement the scope-locked plan
agent: build
---

Implement the plan for: $ARGUMENTS

Read the plan file first (check ~/.opencode/plan/).

Soft gate: if the plan contains no `## Scope Lock` section, stop
immediately and tell me to run /scope first — do not implement anything.

If the lock exists, implement exactly the locked scope, verify by running
the changed feature yourself (per AGENTS.md), and close with: what you
tested, the exact steps used, and the evidence observed.
```

Why commands instead of manual switching: a command with `agent:` frontmatter
switches the CURRENT session's agent and keeps all conversation context, so
all three stages see each other's work in one continuous session. Subagents
were rejected for orchestration because they cannot talk to the user — the
interview must happen in-session.

### 4. Workflow section appended to `~/.config/opencode/AGENTS.md`

```md
# Plan → Scope → Build pipeline

Work flows through three stages in one session. Each command switches the
session's primary agent and keeps all prior context:

1. `/plan <task>` — builtin `plan` agent drafts the plan. Draft only, never final.
2. `/scope <task>` — the custom `scope` agent (global, read-only except plan
   files) loads the `grilling` skill and interviews the user in numbered
   rounds until scope is locked, then appends `## Scope Lock` (first line
   `Status: locked`) to the plan file.
3. `/build <task>` — `build` agent implements the locked plan.

Rules:

- No implementation without a `## Scope Lock` section. On `/build` with no
  lock, stop and send the user back to `/scope`.
- The scope stage asks, never assumes: priorities, CAP-style tradeoffs,
  constraints, non-goals, failure modes, acceptance criteria.
- Only `plan` and `scope` agents write plan files; nobody edits code during
  the scope stage.
- If no plan draft exists, `/scope` sends the user to `/plan` first.
```

(The pre-existing "Verify before claiming completion" content above this
section was left untouched.)

## Replication checklist

1. `npx skills add mattpocock/skills@grill-me -g -y`
2. `npx skills add mattpocock/skills@grilling -g -y`
3. Create `~/.config/opencode/agents/scope.md` (content above)
4. Create `~/.config/opencode/commands/{plan,scope,build}.md` (content above)
5. Append the pipeline section to `~/.config/opencode/AGENTS.md`
6. Verify: restart/let OpenCode hot-reload, then run `/plan <task>` →
   `/scope <task>` → `/build <task>` in one session.

## Verification log (2026-10-17)

All stages verified by execution (Tier 1, functional; 0 screenshots), via the
HTTP API on a scratch testbed at
`/private/var/folders/.../T/opencode/pipeline-test` containing a 1-line
`app.py` (`print('ok')`).

Registration (survived a mid-test server restart):

```sh
opencode debug agents        # contains: scope (primary)
opencode api get /api/command  # contains: plan, scope, build
opencode api get /api/skill    # contains: grill-me, grilling
```

Observed behavior:

1. **`/build` soft gate PASS.** New session, `POST
   /api/session/{id}/command {name:"build",...}` with no plan on disk →
   agent replied "No Scope Lock found… I must stop… run `/scope` first";
   session diff empty, `app.py` untouched (verified with `GET .../diff`).
2. **`/plan` PASS.** Wrote `~/.opencode/plan/add-health-endpoint-app-py.md`
   marked `Status: DRAFT`, deferred all decisions to the scope stage, ended
   with the "Draft ready. Run /scope…" handoff.
3. **`/scope` PASS.** Switched session agent to `scope`, loaded the
   `grilling` skill (tool call recorded), interviewed in numbered rounds
   (Round 1: Q1–Q6, Round 2: Q7–Q12, Round 3: Q13–Q14), each question with
   a recommended answer; explicitly asked the CAP-style liveness/readiness
   question; looked up codebase facts itself; **no Scope Lock until the user
   confirmed** (grep count 0 after rounds 1–2).
4. **Scope Lock PASS.** After final answers, `## Scope Lock` (line 104) +
   `Status: locked` (line 106) written with decisions, priority order,
   non-goals, failure-mode decisions, and a 9-point acceptance checklist.
5. **`/build` happy path PASS.** Implemented the locked scope only
   (stdlib server, 83 lines), then executed all 9 acceptance criteria itself
   with cited evidence (curl outputs, od byte check, exit codes, stderr
   logs) and reported in the AGENTS.md format incl. tier/why/screenshot
   count. Final `pgrep`/`lsof`: no strays, port 8000 free.

### Lessons learned (matters for replication)

- **The gate lives in the command template, not the agent.** Invoking
  `opencode run --agent build "…"` (raw message, bypassing `/build`) does
  NOT carry the gate text; that run implemented without a lock. AGENTS.md's
  "no implementation without Scope Lock" rule alone was not enough for the
  default model on a direct user instruction. Always drive the pipeline via
  the `/plan` → `/scope` → `/build` commands.
- **Sessions survive service restarts** (plan/scope state and the lock file
  persisted through one restart mid-build); re-issue the command after a
  restart if it was cancelled.
- **Provider stalls happen** (Nvidia upstream 503s show in
  `~/.local/share/opencode/log/opencode.log`): long runs can go silent for
  >5 min between steps. `POST /api/session/{id}/interrupt` then a follow-up
  prompt resumes cleanly — the interrupted build finished its remaining
  checks on resume.
- **`grill-me` is a wrapper**; the loadable method is the `grilling` skill
  (its body says `Call the Skill tool with "grilling"`). Install both.

## Not included (deferred)

- Hard gate plugin/hook (enforcing the lock even if the model forgets) —
  current gate is prompt-level ("soft gate") in the build command.
- A custom CAP/engineering-checklist skill — folded into the scope agent's
  system prompt instead; `grilling` supplies the interview method.
- Per-project overrides — everything is global; drop a project-local
  `.opencode/agents/` or `.opencode/commands/` file to override per repo.
