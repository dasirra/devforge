# Building Contract Examples And Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every `/forge:building` contract criterion a concrete worked example, make the Phase 3 contract gate stop reliably on every harness, and add a cheap builder self-check before handoff — with no new tooling, no new harness-binding rows, and no TDD mandate for labor-tier builders.

**Architecture:** This is a documentation-only change: every edit lands in `skills/building/SKILL.md`, a harness-neutral prose skill file with no build step and no automated test runner. There is no code to compile or execute, so each task's "test" is a grep-verified before/after check on the exact text, plus a final full read-through for coherence. `CHANGELOG.md` and the two version manifests get one closing task.

**Tech Stack:** Markdown only. No dependencies, no build tooling.

## Global Constraints

- Never use the em dash symbol anywhere written; restructure the sentence instead (user's global writing rule).
- Preserve the file's existing harness-neutral vocabulary exactly: the phrase `an interactive question` must appear verbatim wherever a hard-stop human interaction is required, because `docs/harness-bindings/claude-code.md` binds it via exact-string substitution and a round-trip check depends on the phrase being unchanged.
- No new rows in any `docs/harness-bindings/*.md` file — this plan's design explicitly reuses the existing `an interactive question` binding.
- Match the existing prose style of `skills/building/SKILL.md`: bold lead-ins on new instructions, `**word**` for emphasis, no headers deeper than `###` are used elsewhere so don't introduce any.
- Version bump follows this repo's semver convention observed in `CHANGELOG.md` (2.0.0 -> 2.1.0 -> 2.1.1): a behavioral addition to a skill is a minor bump. This change bumps `2.1.1` -> `2.2.0`.

---

## Task 1: Add the `example` field to the contract criterion schema

**Files:**
- Modify: `skills/building/SKILL.md` (the "State on disk" section, near the top of the file)

**Interfaces:**
- Produces: the `example` field name and its meaning, which Tasks 2-4 all reference by that exact name.

- [ ] **Step 1: Confirm the current schema text**

Run:
```bash
grep -n "Each criterion is" -A 1 skills/building/SKILL.md
```
Expected output (current state, no `example` field):
```
  Each criterion is
  `{id, issue, criterion, verify_how, status: proposed|agreed|pass|fail}`.
```

- [ ] **Step 2: Edit the schema line**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
  Each criterion is
  `{id, issue, criterion, verify_how, status: proposed|agreed|pass|fail}`.
  `issue` is the issue number the criterion belongs to, or `"integration"`
  for cross-issue behavior. Single-issue and free-form runs use one issue
  value throughout.
```

new_string:
```
  Each criterion is
  `{id, issue, criterion, example, verify_how, status: proposed|agreed|pass|fail}`.
  `example` is a single worked-example string showing the criterion applied
  to concrete data, in the same register as the criterion itself (for
  example, `parse('') -> raises ValueError('empty input')`). `issue` is the
  issue number the criterion belongs to, or `"integration"` for cross-issue
  behavior. Single-issue and free-form runs use one issue value throughout.
```

- [ ] **Step 3: Verify the edit landed**

Run:
```bash
grep -n "example.*verify_how\|worked-example string" skills/building/SKILL.md
```
Expected: two matching lines, the schema tuple and the field description.

- [ ] **Step 4: Commit**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): add example field to contract criterion schema"
```

---

## Task 2: Require the generator to write `example` for every criterion

**Files:**
- Modify: `skills/building/SKILL.md` (Phase 2 step 1, "Generator proposes")

**Interfaces:**
- Consumes: the `example` field name from Task 1.
- Produces: the generator-facing instruction Task 3's mechanical check enforces.

- [ ] **Step 1: Confirm the current instruction text**

Run:
```bash
grep -n "Never \"movement works\"" skills/building/SKILL.md
```
Expected: one match, inside the criteria-writing paragraph of Phase 2 step 1.

- [ ] **Step 2: Edit the criteria-writing paragraph**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
   `cli`: "passing no arguments prints usage to stderr and exits 2". Never
   "movement works", never "handles empty input gracefully". Cover the
   unhappy paths the issue's Proposed behavior section describes: empty
   states, errors, edge cases.
```

new_string:
```
   `cli`: "passing no arguments prints usage to stderr and exits 2". Never
   "movement works", never "handles empty input gracefully". Cover the
   unhappy paths the issue's Proposed behavior section describes: empty
   states, errors, edge cases.

   Alongside `verify_how`, write `example` for every criterion: one concrete
   worked case applying the criterion to specific data, e.g. for the
   ArrowLeft criterion above, `player at (3,3), press ArrowLeft -> player at
   (2,3)`. A criterion with no `example` is incomplete, not merely terse.
```

- [ ] **Step 3: Verify the edit landed**

Run:
```bash
grep -n "A criterion with no \`example\` is incomplete" skills/building/SKILL.md
```
Expected: one match.

- [ ] **Step 4: Commit**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): instruct generator to write an example per criterion"
```

---

## Task 3: Reject criteria with a missing example (lead's mechanical check) and add it to the evaluator's attack list

**Files:**
- Modify: `skills/building/SKILL.md` (Phase 2 step 1.5, "Lead verifies grounding mechanically"; Phase 2 step 2, "Evaluator attacks")

**Interfaces:**
- Consumes: the `example` field name from Task 1 and the generator instruction from Task 2.

- [ ] **Step 1: Confirm the current step 1.5 text**

Run:
```bash
grep -n "A token with no entry blocks the relay" skills/building/SKILL.md
```
Expected: one match.

- [ ] **Step 2: Edit step 1.5 to reject missing examples**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
   fixtures and every `verify_how` for substrate-shaped tokens (ALL_CAPS
   identifiers, slash paths, collection and table names, URLs, package names)
   and confirm each appears in `grounding`. A token with no entry blocks the
   relay.
```

new_string:
```
   fixtures and every `verify_how` for substrate-shaped tokens (ALL_CAPS
   identifiers, slash paths, collection and table names, URLs, package names)
   and confirm each appears in `grounding`. A token with no entry blocks the
   relay. A criterion with an empty or missing `example` is also rejected
   back to the generator, the same way an EXISTS entry with no grep hit is.
```

- [ ] **Step 3: Verify step 1.5's edit landed**

Run:
```bash
grep -n "rejected.*back to the generator, the same way an EXISTS entry" skills/building/SKILL.md
```
Expected: one match.

- [ ] **Step 4: Confirm the current step 2 text**

Run:
```bash
grep -n "tests that would pass while the feature" skills/building/SKILL.md
```
Expected: one match, inside the "Evaluator attacks" bullet.

- [ ] **Step 5: Edit step 2 to add the example-mismatch finding**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
2. **Evaluator attacks** (judgment-tier, sees ONLY the issue body and the proposed
   contract): find missing edge cases, criteria too vague to verify, scope
   beyond the issue, criteria tagged to the wrong issue or integration
   behavior missing entirely, tests that would pass while the feature
   is broken, and **ungrounded substrate**: any store, path, env var,
```

new_string:
```
2. **Evaluator attacks** (judgment-tier, sees ONLY the issue body and the proposed
   contract): find missing edge cases, criteria too vague to verify, scope
   beyond the issue, criteria tagged to the wrong issue or integration
   behavior missing entirely, tests that would pass while the feature
   is broken, an `example` that contradicts or fails to illustrate its own
   criterion, and **ungrounded substrate**: any store, path, env var,
```

- [ ] **Step 6: Verify step 2's edit landed**

Run:
```bash
grep -n "contradicts or fails to illustrate its own" skills/building/SKILL.md
```
Expected: one match.

- [ ] **Step 7: Commit**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): enforce examples via mechanical check and evaluator attack"
```

---

## Task 4: Rewrite Phase 3 with a mandated gate template and a real hard stop

**Files:**
- Modify: `skills/building/SKILL.md` (Phase 3, "Human gate on the contract")

**Interfaces:**
- Consumes: the `example` field from Task 1, and the bound neutral phrase `an interactive question` (already defined in every `docs/harness-bindings/*.md`, no changes needed there).
- Produces: the gate template and the Approve/Request changes/Abort options, which are this task's own deliverable (nothing later depends on their exact wording).

- [ ] **Step 1: Confirm the current Phase 3 text**

Run:
```bash
grep -n "^## Phase 3" skills/building/SKILL.md
```
Expected: one match, giving the line number to anchor the read below.

- [ ] **Step 2: Read the full current Phase 3 section for context**

Run:
```bash
sed -n '/^## Phase 3/,/^## Phase 4/p' skills/building/SKILL.md
```
Expected: the full current section (7 paragraphs), matching the text quoted in Step 3's old_string below.

- [ ] **Step 3: Replace Phase 3 with the templated, hard-stopping version**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
## Phase 3: Human gate on the contract (default)

Default: pause. Show the user the agreed contract, in this order: the
grounding block (NEW entries first), the evaluator's residual risks, then the
criteria list and any disputes. Wait for approval or adjustments before
building. A human skims what he is shown first; show him the premises, not the
paperwork.

The pause is the default because the contract is the last artifact a human can
correct cheaply. After it, every criterion, every builder, and every sibling
issue inherits its premises, and the amendment rule makes changing one an
event. A minute here is worth a rebuild there.

If `--no-gate` was passed: no pause. The agreed contract is already posted to
the issue (Phase 2), leading with the same grounding block and residual risks
above the criteria, so the human can inspect it asynchronously and the run
continues into execution. Use this when you already trust the premises, not to
save a minute.

Either way, the NEW-substrate escalation in Phase 2 step 1.5 is not part of
this gate and fires regardless. It asks whether the contract may invent a
store; this gate asks whether the contract is right.
```

new_string:
```
## Phase 3: Human gate on the contract (default)

Default: pause. Render the agreed contract in this fixed template, one visual
unit per criterion, statement plus example plus verification method:

```
Grounding
  NEW:
    - <name>: <evidence or "not yet verified">
  EXISTS:
    - <name>: <file:line>

Residual risks (from evaluator)
  - <risk 1>
  - ...

Criteria, grouped by issue
  #<issue> — <title>
    [<id>] <criterion>
           example: <example>
           verify: <verify_how>
    ...
  Disputes (if any): <criterion id> — GENERATOR: <reasoning> / EVALUATOR: <objection>
```

Grounding leads (NEW entries first) because it is the premise everything else
stands on; a human skims what he is shown first. Then ask an interactive
question: "Approve this contract?" with options **Approve (Recommended)**,
**Request changes**, and **Abort**. Do not proceed to Phase 4 until the answer
is Approve.

The pause is the default because the contract is the last artifact a human can
correct cheaply. After it, every criterion, every builder, and every sibling
issue inherits its premises, and the amendment rule makes changing one an
event. A minute here is worth a rebuild there.

If `--no-gate` was passed: no pause, and no interactive question. The agreed
contract is already posted to the issue (Phase 2), rendered in the same
template above, so the human can inspect it asynchronously and the run
continues into execution. Use this when you already trust the premises, not to
save a minute.

Either way, the NEW-substrate escalation in Phase 2 step 1.5 is not part of
this gate and fires regardless. It asks whether the contract may invent a
store; this gate asks whether the contract is right.
```

- [ ] **Step 4: Verify the edit landed and the bound phrase is intact**

Run:
```bash
grep -n "an interactive question" skills/building/SKILL.md
```
Expected: this now matches in Phase 0.5 (surface question), Phase 2 step 1.5 (NEW-substrate escalation), and the new Phase 3 — three matches total, all using the exact phrase `an interactive question`.

Run:
```bash
sed -n '/^## Phase 3/,/^## Phase 4/p' skills/building/SKILL.md | grep -c "example:"
```
Expected: `1` (the template's `example:` line).

- [ ] **Step 5: Round-trip check against the Claude Code binding**

Read `docs/harness-bindings/claude-code.md` and manually substitute its table
into the new Phase 3 text: every occurrence of `an interactive question`
becomes `AskUserQuestion`, `judgment-tier`/`labor-tier` stay as `opus`/
`sonnet` elsewhere in the file unaffected by this task. Confirm the
substituted sentence still reads correctly: "...then ask AskUserQuestion:
'Approve this contract?'..." — grammatically odd but semantically exact,
matching how the same substitution already reads at the Phase 0.5 and Phase 2
step 1.5 call sites elsewhere in the file. This is the same round-trip
property `docs/harness-bindings/README.md` describes; no binding-doc edit is
needed because no new phrase was introduced.

- [ ] **Step 6: Commit**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): mandate a gate template and end Phase 3 in a hard stop"
```

---

## Task 5: Add the Phase 4 builder self-check

**Files:**
- Modify: `skills/building/SKILL.md` (Phase 4, "Execution")

**Interfaces:**
- Consumes: the `example` and `verify_how` fields from Task 1.

- [ ] **Step 1: Confirm the current Phase 4 bullet list**

Run:
```bash
grep -n "Forbid this run's vocabulary in the artifact" skills/building/SKILL.md
```
Expected: one match, the last bullet in Phase 4's list before "Cross-workstream integration defects...".

- [ ] **Step 2: Edit Phase 4 to add the self-check bullet**

Use the Edit tool on `skills/building/SKILL.md` with:

old_string:
```
- Each teammate appends what it did to `harness/progress.json`.
- Keep changes consistent with surrounding code: naming, structure, comment
  density.
- **Forbid this run's vocabulary in the artifact, in every teammate prompt.**
```

new_string:
```
- Each teammate appends what it did to `harness/progress.json`.
- Keep changes consistent with surrounding code: naming, structure, comment
  density.
- **Before reporting done, each teammate exercises every criterion in its
  brief against its own build**, the way that criterion's `example` and
  `verify_how` describe, and fixes what fails first. This is a pre-handoff
  check, not a build-order rule: it does not change when code gets written
  and it does not replace Phase 6. The evaluator still never sees this
  check, never reads builder tests or implementation, and still writes its
  own scripts from the contract alone; this step only reduces how much
  obviously-broken work reaches that loop.
- **Forbid this run's vocabulary in the artifact, in every teammate prompt.**
```

- [ ] **Step 3: Verify the edit landed**

Run:
```bash
grep -n "exercises every criterion in its" skills/building/SKILL.md
```
Expected: one match.

- [ ] **Step 4: Commit**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): add a builder self-check before handoff"
```

---

## Task 6: Full read-through for coherence

**Files:**
- Read only: `skills/building/SKILL.md` (no modification expected; this task is a verification gate, not an editing task)

**Interfaces:**
- Consumes: the combined output of Tasks 1-5.

- [ ] **Step 1: Read the whole file top to bottom**

Run:
```bash
cat -n skills/building/SKILL.md
```

Check, in order:
1. The "State on disk" schema (Task 1) matches every later reference to
   `example` in Phase 2 (Task 2, Task 3) — same field name, no drift like
   `example` vs `worked_example`.
2. Phase 3's template (Task 4) renders `example:` using the same field the
   schema defines, not a paraphrase.
3. Phase 4's self-check bullet (Task 5) references `example` and
   `verify_how`, both defined in the Task 1 schema.
4. No leftover placeholder text ("TBD", "TODO") was introduced by any edit.
5. Every occurrence of `an interactive question` (three total, per Task 4
   Step 4) is spelled identically, since the Claude Code binding is an
   exact-string substitution.

- [ ] **Step 2: If any inconsistency is found, fix it directly with the Edit tool, re-run the Step 1 read, and confirm it is resolved**

(No code sample here: this step only fires conditionally, and the fix depends
on what Step 1 finds. If Step 1 finds nothing, skip straight to Step 3.)

- [ ] **Step 3: Commit only if Step 2 made a fix**

```bash
git add skills/building/SKILL.md
git commit -m "docs(building): fix coherence issue found in full read-through"
```

If Step 2 made no fix, there is nothing to commit; proceed to Task 7.

---

## Task 7: Changelog and version bump

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `package.json`
- Modify: `.claude-plugin/plugin.json`

**Interfaces:**
- Consumes: nothing from earlier tasks besides their existence as shipped changes to summarize.

- [ ] **Step 1: Confirm current versions**

Run:
```bash
grep -n "\"version\"" package.json .claude-plugin/plugin.json
grep -n "^## \[" CHANGELOG.md | head -1
```
Expected: both files show `"version": "2.1.1"`, and the changelog's top entry is `## [2.1.1] - 2026-07-17`.

- [ ] **Step 2: Add the new changelog entry**

Use the Edit tool on `CHANGELOG.md` with:

old_string:
```
# Changelog

All notable changes to this project will be documented in this file.

## [2.1.1] - 2026-07-17
```

new_string:
```
# Changelog

All notable changes to this project will be documented in this file.

## [2.2.0] - 2026-07-17

### Added
- Every `/forge:building` contract criterion now carries an `example` field: a concrete worked case (e.g. `parse('') -> raises ValueError('empty input')`), required by the generator, rejected if missing by the lead's mechanical check, and attacked by the evaluator if it contradicts its criterion.
- The Phase 3 contract gate now renders a fixed template (grounding, residual risks, criteria grouped by issue with their example and verification method) and ends in a real interactive-question hard stop, so the gate reliably pauses on every harness instead of depending on prose alone.
- Labor-tier builders now self-check every criterion in their brief against their own build, using its example and verification method, before reporting done — cutting wasted Phase 6 evaluation rounds without weakening the evaluator's independence.

## [2.1.1] - 2026-07-17
```

- [ ] **Step 3: Bump the version in both manifests**

Use the Edit tool on `package.json` with:

old_string:
```
  "version": "2.1.1",
```

new_string:
```
  "version": "2.2.0",
```

Use the Edit tool on `.claude-plugin/plugin.json` with:

old_string:
```
  "version": "2.1.1",
```

new_string:
```
  "version": "2.2.0",
```

- [ ] **Step 4: Verify all three files agree**

Run:
```bash
grep -n "\"version\"" package.json .claude-plugin/plugin.json
grep -n "^## \[" CHANGELOG.md | head -1
```
Expected: both manifests show `"version": "2.2.0"`, and the changelog's top entry is `## [2.2.0] - 2026-07-17`.

- [ ] **Step 5: Commit**

```bash
git add CHANGELOG.md package.json .claude-plugin/plugin.json
git commit -m "chore: bump forge to 2.2.0"
```

---

## Self-Review Notes

**Spec coverage:** Task 1 covers the spec's `example` field (Section 1). Tasks 2-3 cover Section 1's enforcement (generator writes it, lead rejects missing, evaluator attacks mismatches). Task 4 covers Section 2 in full (template + hard stop via `an interactive question`, no new binding rows). Task 5 covers Section 3 (builder self-check, explicitly not TDD). The spec's "explicitly not adopted" TDD decision required no skill-file edit — it is a decision recorded in the spec, not an instruction to add — so no task implements it; Task 6 confirms nothing in the diff accidentally implies otherwise. Task 7 covers the repo's own changelog/version convention, observed in `CHANGELOG.md` and required to keep `package.json` and `.claude-plugin/plugin.json` in sync as prior commits did.

**Placeholder scan:** No TBD/TODO introduced; every new sentence is complete, concrete text ready to commit as-is.

**Type consistency:** The field name `example` is identical everywhere it appears (Task 1's schema, Task 2's generator instruction, Task 3's mechanical check and evaluator attack, Task 4's gate template, Task 5's self-check bullet, Task 6's cross-check). `verify_how` is unchanged throughout, reused rather than renamed.
