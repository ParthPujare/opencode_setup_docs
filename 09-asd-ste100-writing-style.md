# Change: ASD-STE100 Writing Style for All Agents

**Date:** 2026-10-18
**Files Created:** `~/.config/opencode/references/asd-ste100-writing.md`,
`~/.config/opencode/agents/plan.md`
**Files Modified:** `~/.config/opencode/agents/scope.md`,
`~/.config/opencode/agents/build.md`,
`~/.config/opencode/agents/verify.md`
**Follows:** `plan-scope-build-pipeline.md` (the scope/build agents this change extends)

---

## Summary

All four pipeline agents — `plan`, `scope`, `build`, `verify` — now write their
user-facing prose in ASD-STE100 Simplified Technical English. STE is the
aerospace standard (Issue 9, January 2025) for controlled technical language:
53 writing rules and a ~900-word approved dictionary, designed so a reader with
basic English understands every sentence on first read.

The change is a **prose-only** policy. It governs what each agent writes *for
the user*: plans, interview questions, scope locks, test reports, findings, and
recommended fixes. It does **not** govern code, code comments, commit messages,
variable names, file paths, command lines, or quoted code snippets — those keep
the project's own conventions. The `build` agent carries this exemption most
explicitly, and it was rechecked twice before delivery.

---

## Why

Agent output is technical documentation. The four failure modes STE was built
to eliminate appear constantly in raw LLM output:

1. **Long, compound sentences.** A single 35-word sentence carrying three ideas
   joined by semicolons.
2. **Passive voice.** "The tests should be run" hides who runs them.
3. **Nominalizations and vague verbs.** "Utilize," "leverage," "facilitate,"
   "in order to" — longer words that slow the reader and add no meaning.
4. **Inconsistent terminology.** Calling the same component "the validator,"
   "the middleware," and "the check layer" across one report.

STE's rules fix all four with hard, checkable constraints: a 20/25-word
sentence cap, mandatory active voice, an approved-verb table, and a
consistent-terminology rule.

---

## What ASD-STE100 is

ASD-STE100 (formerly AECMA Simplified English) is an international standard
maintained by the ASD STE Maintenance Group. It has two parts:

- **Part 1 — Writing rules.** 53 rules across 9 sections. They cover word
  choice, verb forms, sentence length, procedural writing, descriptive writing,
  safety instructions, and punctuation.
- **Part 2 — Dictionary.** ~900 approved words. Each has one approved meaning
  and one part of speech. Synonyms are excluded: the dictionary approves
  "start," never "begin," "commence," or "initigate."

The rules most relevant to agent output:

| Rule | Constraint |
|------|-----------|
| 5.1 / 6.3 | Max 20 words per procedural sentence, 25 per descriptive sentence |
| 3.6 | Active voice; passive only when the doer is truly unknown |
| 5.3 | Imperative mood for all instructions ("Run the tests") |
| 5.4 | Conditions first ("When X, do Y") |
| 5.2 | One instruction per sentence |
| 6.5 / 6.6 | One topic per paragraph, max 6 sentences per paragraph |
| 4.3 | Vertical lists for any 3+ items |
| 8.1 | No semicolons |
| 4.2 | No contractions |
| GR-6 | No Latin abbreviations ("for example," not "e.g.") |
| 9.3 | No phrasal verbs where a single verb exists |
| 3.5 | No progressive "-ing" verb forms |
| 9.4 | Consistent terminology per concept |
| 1.14 | American English spelling |

---

## The prose/code boundary

The one design decision that needed explicit enforcement: STE governs prose,
never code. Left implicit, an agent could rewrite a code comment into STE form
or rename a variable to an "approved" word. Both would be wrong — code follows
the project's own style, not a writing standard.

Three mechanisms enforce the boundary:

1. **Each agent's Writing style section opens with a bold exemption.** The
   `build` and `verify` agents state it as a standalone bold line: "This
   section governs your prose output only. It does not govern code."
2. **Each section carries an explicit exclusion list.** Source code, inline
   comments, docstrings, variable names, function names, class names, file
   names, commit messages, quoted code snippets, file paths, command lines,
   terminal output.
3. **The `build` and `verify` agents end with a double-check instruction.**
   Before finishing, the agent re-reads its report and confirms no STE rule
   leaked into code. If a comment or variable name was rewritten into STE form,
   it reverts it to the project's convention.

The `build` agent's exemption was rechecked twice before delivery, per the
user's explicit instruction. Recheck 1 read the full file and confirmed every
STE rule is scoped to "your report," "test steps," "evidence observed," or
"blockers" — none mentions code. Recheck 2 grepped the shared reference file
for `code|comment|commit|variable|function name|snippet` and found exactly two
matches, both safe: the reference file's own header exemption, and a
word-replacement table entry that governs prose word choice, not code.

---

## What was changed

### 1. New shared reference: `~/.config/opencode/references/asd-ste100-writing.md`

A distilled, agent-output-focused adaptation of the 53 rules — not a copy of
the full ~200-page standard. It contains:

- The 13 core rules above, each with a Non-STE → STE example pair adapted to
  software and agent context.
- An approved-verb cheat sheet for common agent actions: `use` (not "utilize"),
  `run` (not "execute"), `start` (not "commence"), `stop` (not "terminate"),
  and 15 more.
- A word-replacement table: "prior to" → "before," "in order to" → "to,"
  "a number of" → "some," "functionality" → "function," and so on.
- An American English spelling table.
- A 13-step silent self-check the agent runs before presenting any output.

The file header states the exemption up front: "It does not govern code, code
comments, commit messages, file paths, or command lines — those follow the
project's own conventions."

This mirrors the existing pattern where `scope.md` reads
`references/engineering-principles.md` as a shared rubric.

### 2. New custom agent: `~/.config/opencode/agents/plan.md`

Replaces the builtin `plan` agent. The builtin was referenced by
`commands/plan.md` via `agent: plan` and could not be edited. A custom agent at
this path overrides it by the same name; `commands/plan.md` is unchanged and
now resolves to the custom agent.

The custom agent keeps the builtin's job — explore the codebase, write a draft
plan to `~/.opencode/plan/`, end with the "Draft ready. Run /scope…" handoff —
and adds a Writing style section scoped to the plan document: Goal, Context,
Proposed approach, Steps, Risks and open questions, Acceptance criteria.

### 3. Edited agent: `scope.md`

Appended a Writing style section after the existing Rules. STE applies to
interview questions, recommended answers, the classification statement, skip
justifications, and the entire `## Scope Lock` section. One agent-specific rule
was added on top of the shared set: each `SKIPPED` line in the Applicability
subsection is one sentence of at most 25 words.

### 4. Edited agent: `build.md`

Appended a Writing style section with the strongest prose/code separation of
the four. STE applies to the final report only. The section opens with the
bold exemption, lists 13 excluded surfaces, states "Write code the way the
project's own conventions require. Write your report the way STE requires.
These are two separate surfaces with two separate rules," and closes with the
revert-if-leaked double-check.

### 5. Edited agent: `verify.md`

Appended a Writing style section. STE applies to the findings report. The
`VERDICT:` line is explicitly frozen — "stays exactly as specified in the
Procedure section. Do not reword it." Quoted code, file paths, and terminal
output stay verbatim as the tools produced them.

---

## Before / after

Same task, two styles. Task: "Add input validation to the user registration
endpoint."

### Without STE (typical raw output)

> The user registration endpoint currently doesn't have any input validation,
> which means that invalid or malicious data could end up being stored in the
> database; this is a significant concern for data integrity and security. This
> plan outlines the steps that need to be taken in order to add comprehensive
> input validation to the registration endpoint, ensuring that only valid,
> well-formed data is accepted before it reaches persistence.
>
> **Steps:**
> 1. First, we'll need to install the `zod` validation library, which has been
>    chosen for its schema-based approach; it provides a clean API for defining
>    schemas and parsing untrusted input.
> 2. Once the library has been installed, a validation schema should be created
>    that defines the rules for each field (e.g. email format, password length
>    requirements, username character restrictions, and so on).

Violations: 35-word opening sentence (over the 25-word cap), two semicolons,
two contractions ("doesn't," "we'll"), passive voice ("should be created"),
Latin abbreviation ("e.g."), vague phrasal verb ("end up being stored").

### With STE

> The user registration endpoint accepts data without validation. Invalid data
> can reach the database. This plan adds input validation to the endpoint. The
> endpoint will accept only well-formed data before it writes to the database.
>
> **Steps:**
> 1. Install the `zod` validation library. Use it to define schemas that parse
>    untrusted input.
> 2. Create a validation schema for each registration field. Define rules for
>    email format, password length, and username characters.

Every sentence within its cap. No semicolons, contractions, or Latin
abbreviations. Active voice throughout. Each step is imperative. One idea per
sentence.

---

## Verification

Tier 1 (functional, text-only; 0 screenshots). This is a configuration change
to agent instruction files — there is no runnable app or UI surface, so file
inspection and grep are the correct verification channel.

Executed checks:

1. Read `asd-ste100-writing.md` and confirmed the header exempts code, code
   comments, commit messages, file paths, and command lines.
2. Read `agents/plan.md` and confirmed its closing line exempts code snippets,
   file paths, command lines, and quoted code.
3. Read `agents/scope.md` and confirmed its closing line exempts code
   snippets, file paths, and command lines.
4. Read `agents/build.md` twice. Recheck 1 confirmed the exclusion list covers
   all 13 surfaces and the double-check catches leaks. Recheck 2 confirmed the
   reference file's only two "code" matches are safe.
5. Read `agents/verify.md` and confirmed the exclusion line covers code,
   quoted snippets, file paths, command lines, and terminal output.
6. Grepped the reference file for `code|comment|commit|variable|function
   name|snippet` — exactly two matches, both safe.

File sizes at delivery: `asd-ste100-writing.md` = 10,635 bytes.

---

## Rollback

Remove the Writing style section from each of the four agent files:

```bash
# For each agent, delete the "## Writing style — ASD-STE100" heading
# and everything after it, up to the end of file (the section is always
# appended last):
~/.config/opencode/agents/plan.md
~/.config/opencode/agents/scope.md
~/.config/opencode/agents/build.md
~/.config/opencode/agents/verify.md

# To restore the builtin plan agent, delete the custom override:
rm ~/.config/opencode/agents/plan.md

# Optional: remove the shared reference
rm ~/.config/opencode/references/asd-ste100-writing.md
```

No service restart needed — agent files hot-reload on the next prompt. Deleting
`agents/plan.md` restores the builtin `plan` agent on the next `/plan` invocation,
since `commands/plan.md` references `agent: plan` by name.

---

## Not included (deferred)

- **The full STE dictionary.** The ~900-word approved dictionary is not copied
  into the reference file. The reference carries an approved-verb cheat sheet
  and a word-replacement table for the words that appear most in agent output.
  Agents do not enforce the full dictionary word by word.
- **STE for code comments.** Explicitly excluded by user instruction. Code
  comments follow the project's own conventions.
- **A lint gate for STE compliance.** The self-check inside the reference file
  is agent-run, not tool-enforced. No external STE linter (for example a
  `ste-lint` binary) is wired into the pipeline.
- **Per-agent model pinning.** All four agents inherit the session's model. No
  model change was made for this writing-style work.
