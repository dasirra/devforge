# Design: worked examples on contract criteria, a reliable Phase 3 gate, and a builder self-check

Date: 2026-07-17
Status: approved design, pending spec review

## Problem

Using forge on another project surfaced two related gaps in `/forge:building`:

1. **The Phase 3 contract gate is unreliable across harnesses.** Every other
   mandatory human touchpoint in the skill (the Phase 0.5 surface question, the
   Phase 2 step 1.5 NEW-substrate escalation) is written using the bound
   neutral phrase `an interactive question`, which every harness binding maps
   to a real hard-stop primitive (`AskUserQuestion` on Claude Code, a plain
   chat question on Pi, `default_api:ask_question` on Antigravity). Phase 3,
   the most consequential pause in the pipeline, is written as loose prose
   instead ("pause... show the user... wait for approval"). A harness whose
   agent reads that as "print and continue" is not violating the text; there
   is nothing bound for it to violate. In practice this gate does not reliably
   stop and display on every harness.

2. **Contract criteria lack a forced concrete example.** Forge's best criteria
   already read like worked examples ("`parse('')` raises `ValueError` whose
   message contains `'empty input'`"), but nothing in Phase 2 requires one. A
   vaguer criterion can pass the generator/evaluator exchange without ever
   being pinned to a concrete case, and when the gate does display, there is
   no simple, uniform place to see what each clause is actually about.

A third, related question came up: whether `/forge:building`'s labor-tier
builders should practice test-driven development (write a failing unit test,
then the implementation) as they build against each criterion, the way Robert
Martin describes his own workflow (Gherkin -> acceptance test -> unit test ->
code).

## Goal

Fix both gaps with the smallest mechanism that serves them, and settle the TDD
question, without adding new tooling, new harness-binding rows, or a technique
mandate that contradicts forge's existing design.

## Design

### 1. `example` field on every contract criterion

Add one field to the criterion schema in `harness/contract.json`:

```
{id, issue, criterion, example, verify_how, status}
```

`example` is a single worked-example string, in the same voice forge's
strongest criteria already use:

- `parse('') raises ValueError whose message contains 'empty input'`
  -> example: `parse('') -> raises ValueError('empty input')`
- `pressing ArrowLeft moves the player one cell left`
  -> example: `player at (3,3), press ArrowLeft -> player at (2,3)`

Not a structured Given/When/Then object: a plain string is cheaper for the
generator to write, cheaper for the gate template to render, and matches the
one-liner density the rest of the criteria already have (40-60 criteria in a
multi-issue run should not double in size).

**Enforcement, both mechanical:**

- **Phase 2 step 1 (generator proposes):** the generator writes `example` for
  every criterion in the same pass as the criterion itself.
- **Phase 2 step 1.5 (lead's mechanical check):** a criterion with an empty or
  missing `example` is rejected back to the generator, the same way an EXISTS
  entry with no grep hit is today. The evaluator's adversarial attack (Phase 2
  step 2) gains one more concrete thing to hunt: an `example` that does not
  actually match its criterion's stated behavior.

### 2. Phase 3 gate: fixed template + real hard stop

Two changes to Phase 3.

**A mandated display template**, replacing today's loose ordering instruction:

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

One visual unit per criterion: statement, example, verification method. This
is a prose/template addition to the skill file only; no schema change beyond
Section 1's `example` field, no new tooling.

**End the pause in `an interactive question`.** After rendering the template,
the lead asks an actual interactive question: *"Approve this contract?"* with
options **Approve (Recommended)** / **Request changes** / **Abort**. Because
`an interactive question` is already a bound neutral phrase with a row in
every harness binding doc, this gets every harness a genuine hard stop with no
new binding-doc work.

This applies to the default (gated) path. `--no-gate` is unchanged: the
contract is still posted to the issue for async review, leading with the same
grounding-first ordering, just without the interactive pause.

### 3. Phase 4: builder self-check (not TDD)

One addition to the per-teammate briefing in Phase 4 (Execution):

> Before reporting done, exercise every criterion in your brief against your
> own build, the way its `example` and `verify_how` describe. Fix what fails
> before handing off.

This reuses the `example` field from Section 1 so builders have something
concrete to check against. It is a cheap pre-handoff check, not a build-order
mandate: it doesn't touch when code gets written, doesn't require a test
before the implementation, and doesn't prescribe technique. It reduces how
much obviously-broken work reaches Phase 6, cutting wasted evaluator rounds,
without weakening the separation rule: the evaluator still never sees this
self-check, never reads builder tests or implementation, and still writes its
own scripts from the contract alone.

### On TDD: explicitly not adopted for Phase 4

Decision, recorded here because it was actively debated: `/forge:building`
will **not** instruct labor-tier builders to practice unit-level TDD
(red-green-refactor) as they implement each criterion. Reasoning:

- Forge's generator/evaluator design already delivers the property Robert
  Martin's Gherkin -> acceptance test -> unit test -> code sequence exists to
  guarantee: a spec fixed and verified independently of the implementer,
  before code is trusted. Forge does this adversarially (separate context,
  forbidden from reading the implementation), which is a stronger form of
  independence than a self-authored acceptance test provides.
- The skill already argues, as a guardrail, that builder-written tests
  encode the builder's own understanding, so a green suite certifies whatever
  misunderstanding produced the bug. That argument applies identically whether
  the builder's test is written first or after; TDD ordering does not address
  the failure mode forge is guarding against.
- Phase 4's design principle is that the contract is the plan and technique
  belongs to the builder. Mandating red-green-refactor is exactly the kind of
  upfront technical prescription forge avoids everywhere else in that phase.

The Gherkin question for `/forge:planning` (a separate, PM-altitude-safe
adoption of Given/When/Then for acceptance criteria) is out of scope for this
design and will be brainstormed as its own spec.

## Out of scope

- Any change to `/forge:planning` or `/forge:interview`.
- Full Gherkin adoption (`.feature` files, Cucumber-style step definitions, or
  any executable-BDD tooling) anywhere in forge. Rejected: it would make the
  contract a persistent artifact where forge's design deliberately keeps it
  throwaway (`harness/` gitignored, evaluator scripts never merged), and it
  invites the "features without maintained step definitions" ceremony problem
  without adding anything the `example` field doesn't already give.
- Unit-level TDD mandate for labor-tier builders (see above).
- New harness-binding rows: `an interactive question` already exists in every
  binding doc and is reused as-is.

## Open questions

None outstanding; all three original questions were resolved above.
