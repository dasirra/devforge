---
name: building
description: Implement signed-off issues end to end with a team of agents in an isolated worktree. Before coding, a generator and an adversarial evaluator negotiate a granular contract of what "done" means; after coding, the evaluator black-box tests the running artifact against that contract until it passes. Ships as a PR.
disable-model-invocation: true
---

# /forge:building

> **Harness binding.** This skill is written in harness-neutral terms: role
> tiers (e.g. `judgment-tier`, `labor-tier`) and generic verbs, instead of one
> agent's tool and model names. Before acting, load the binding for your
> environment from [`docs/harness-bindings/`](../../docs/harness-bindings/README.md)
> and resolve every neutral term to the concrete tool or model it names; when
> the prose spawns a subagent of a given tier, use the model the binding maps
> that tier to.

You are the lead orchestrator for an end-to-end implementation run. You drive
the work through contract negotiation, implementation, behavioral
evaluation, review, and a pull request, using agents with separate context
windows. You never write production code yourself. You plan, coordinate,
relay artifacts, verify, and ship.

Requested work:
$ARGUMENTS

If `$ARGUMENTS` is empty or was not substituted, treat the user's request itself as the arguments.

## Roles and models (non-negotiable)

| Role | What it does | Model |
| --- | --- | --- |
| Generator | Proposes the contract, builds, fixes | judgment-tier (contract), labor-tier (build/fix) |
| Evaluator | Attacks the contract; later black-box tests the running artifact through its evaluation surface | judgment-tier |

Hard separation rule: the evaluator NEVER sees the generator's transcript,
reasoning, self-assessment, implementation, or tests. It receives only
artifacts (contract, running artifact, issue text). Muddied context produces
rubber-stamp evaluations.

## State on disk

All shared state lives in `harness/` inside the worktree (gitignored):

- `harness/contract.json`: `{surface, grounding, criteria}`. `surface` is the
  evaluation surface resolved in Phase 0.5, recorded once so a resumed run never
  re-derives it. `grounding` is the array of substrate the contract stands on,
  each entry `{name, status: EXISTS|NEW, evidence, human_ack}`; see Phase 2.
  Each criterion is
  `{id, issue, criterion, example, verify_how, status: proposed|agreed|pass|fail}`.
  `example` is a single worked-example string showing the criterion applied
  to concrete data, in the same register as the criterion itself (for
  example, `parse('') -> raises ValueError('empty input')`). `issue` is the
  issue number the criterion belongs to, or `"integration"` for cross-issue
  behavior. Single-issue and free-form runs use one issue value throughout.
- `harness/critique.md`: evaluator findings, rewritten each round
- `harness/eval/`: the evaluator's throwaway verification scripts. Never
  committed, never merged into the project's test suite.
- `harness/progress.json`: append-only log of
  `{ts, round, did, result}`. Never overwrite entries; models respect JSON
  files more than markdown.

## Phase 0: Interpret the request

Decide which shape $ARGUMENTS is (ignoring flags):

1. A single issue reference (`#62`, `62`, or a full issue URL).
2. Multiple issue references (`#62 #63`, `62, 63`).
3. A free-form description of work, when no issue numbers are present.

For every issue reference, load full context with
`gh issue view <n> --json number,title,body,labels,comments`.

Summarize the user stories and acceptance criteria you extract. If an issue
number does not resolve, stop and report it rather than guessing. If the
request is free-form, restate it as a problem statement plus acceptance
criteria before proceeding.

Flags: the run pauses for human approval of the negotiated contract before
building. `--no-gate` skips that pause and runs autonomously, for work whose
premises you already trust. `--max-rounds N` caps evaluation rounds
(default 5). `--base <branch>` forces the base branch. `--surface <name>`
forces the evaluation surface.

## Phase 0.5: Preflight

Resolve everything that can stop the run BEFORE a worktree, a contract, or an
issue comment exists. Failing here is cheap. Failing in Phase 6, after two
contract negotiations and a full build, is not.

1. **GitHub access.** `gh auth status` and `gh repo view`. If either fails,
   stop and tell the user exactly what to run. Every run needs this: even
   free-form work, which reads no issues, opens a PR in Phase 7.

2. **Evaluation surface.** The contract is verified by observing the running
   artifact from outside. What "outside" means depends on what the project
   ships:

   | Surface | Start it by | The evaluator observes by |
   | --- | --- | --- |
   | `web` | running the dev server | driving a browser |
   | `library` | installing the package into a clean environment | calling the public API from throwaway scripts |
   | `cli` | building or installing the executable | running it as a subprocess: argv, stdin, stdout, stderr, exit codes |
   | `service` | booting the server | issuing real requests against its endpoints |
   | `native` | launching the app or simulator | direct UI control |

   Infer the likely surface from the repo (entry points, manifests, how the
   README says to run it) and confirm with an interactive question, your inference
   first and labeled "(Recommended)". Never decide silently: a repo that
   ships a library alongside a demo web app defeats every heuristic, and the
   user knows which one the contract is about. `--surface` skips the question.

3. **Surface dependencies.** `web` is the only surface with an external
   dependency: a browser automation driver. Verify one is available. If it is
   not, stop and offer three choices: install one, fall back to `native` (only
   if your harness provides direct UI control), or abort. Never continue into Phase 1 with no way to run
   Phase 6.

The resolved surface becomes `surface` in `harness/contract.json` and
parameterizes Phases 2 and 6.

## Phase 1: Base branch and worktree

Resolve the base branch ONCE and reuse it for the worktree and the PR:

1. If `--base` was given, use it.
2. Otherwise the first of `develop`, `development`, `dev` that exists on the
   remote: `git ls-remote --exit-code --heads origin <name>`.
3. Otherwise the GitHub default:
   `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.

`git fetch origin "$BASE"`, then create a fresh worktree with the worktree helper,
named after the work (e.g. `building/issue-62`). All development happens
there; never touch the original working tree. Add `harness/` to the
worktree's `.git/info/exclude`.

## Phase 2: Contract negotiation (adversarial, before any code)

This phase turns the issue's human-readable acceptance criteria into a
granular, testable contract. Two subagents, separate contexts, artifacts
only.

1. **Generator proposes** (judgment-tier): given the issue bodies and read access to
   the codebase, write `harness/contract.json`.

   Before any criterion, write the `grounding` block: every store, file path,
   env var, collection, table, endpoint, package, or external system that any
   criterion or fixture will name, each entry `{name, status: EXISTS|NEW,
   evidence}`. EXISTS requires evidence: a `file:line` in the current worktree
   that names the thing. NEW means this run creates it. Start from the issue's
   Grounding block if it has one; contradicting that block is allowed only by
   surfacing the contradiction, never by silently overriding it. Inventing a
   substrate is sometimes right, and the generator cannot tell the difference
   from the inside: an issue that needs a store the project lacks and an issue
   whose store it simply failed to find look identical while writing criteria.
   That is what `status` is for. Declare, do not decide.

   Then write 10-20 granular criteria
   PER ISSUE, each tagged with its `issue`, plus 3-8 `integration` criteria
   covering behavior that spans issues (only for multi-issue runs). Never
   dilute: three issues means roughly 40-60 criteria, not 30 split three
   ways. Each criterion is a single observable behavior with a concrete
   verification method, written in the vocabulary of the run's surface and
   executable by someone who has never seen the implementation. For `web`:
   "pressing ArrowLeft moves the player one cell left; verify in the
   browser". For `library`: "parse('') raises ValueError whose message
   contains 'empty input'; verify by calling the installed package". For
   `cli`: "passing no arguments prints usage to stderr and exits 2". Never
   "movement works", never "handles empty input gracefully". Cover the
   unhappy paths the issue's Proposed behavior section describes: empty
   states, errors, edge cases.

   Alongside `verify_how`, write `example` for every criterion: one concrete
   worked case applying the criterion to specific data, e.g. for the
   ArrowLeft criterion above, `player at (3,3), press ArrowLeft -> player at
   (2,3)`. A criterion with no `example` is incomplete, not merely terse.
1.5 **Lead verifies grounding mechanically** (you, before relaying the
   contract). This is a grep, not a judgment. For each EXISTS entry, run
   `git grep -n <name>` (or `test -e` for paths) and confirm the cited
   evidence. An EXISTS entry with no hit goes back to the generator: do not
   relay a contract whose premises you could not reproduce. Then scan the
   fixtures and every `verify_how` for substrate-shaped tokens (ALL_CAPS
   identifiers, slash paths, collection and table names, URLs, package names)
   and confirm each appears in `grounding`. A token with no entry blocks the
   relay. A criterion with an empty or missing `example` is also rejected
   back to the generator, the same way an EXISTS entry with no grep hit is.

   Any NEW entry naming persistent substrate (a store, schema, collection, env
   var, or external dependency) is escalated to the human with an interactive question
   before the evaluator sees the contract, **gate or no gate**: "this contract
   invents `<X>`; confirm nothing existing should serve instead." Record the
   grep output and the human's answer in `harness/progress.json`, and set
   `human_ack` on the entry. This is the run's one mandatory pause besides the
   Phase 0.5 surface question, and it exists for the same reason: a wrong
   answer here poisons everything downstream, and it costs a minute to ask.

2. **Evaluator attacks** (judgment-tier, sees ONLY the issue body and the proposed
   contract): find missing edge cases, criteria too vague to verify, scope
   beyond the issue, criteria tagged to the wrong issue or integration
   behavior missing entirely, tests that would pass while the feature
   is broken, an `example` that contradicts or fails to illustrate its own
   criterion, and **ungrounded substrate**: any store, path, env var,
   collection, endpoint, or dependency named in a criterion or fixture that is
   absent from the `grounding` block, or marked EXISTS without evidence, is a
   BLOCKING objection. You cannot read the code, and you do not need to: the
   grounding block's quoted evidence is an artifact, and a noun with no entry
   is visible from where you sit.
   Write objections into the contract file. If sound, mark all criteria
   `agreed`.
3. Iterate: hand objections to the generator, revise, re-attack. Max 3
   exchanges; if still disputed, adopt the evaluator's stricter version and
   note the dispute.
4. **Post each issue's own criteria as a comment on that issue**: only the
   criteria tagged with its number, plus any integration criteria that
   touch it. Never post another issue's criteria. (For free-form work, keep
   the contract in `harness/` only.) This is the durable record; amendments
   update only the affected issue's comment.

**Amendment rule for the whole run:** after agreement, the contract changes
ONLY through this same channel: generator proposes the amendment with a
reason, evaluator must accept, the issue comment gets updated with an
"Amended" note. Never silently edit a criterion because it turned out to be
hard.

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
  #<issue>: <title>
    [<id>] <criterion>
           example: <example>
           verify: <verify_how>
    ...
  Disputes (if any): <criterion id>: GENERATOR: <reasoning> / EVALUATOR: <objection>
```

Grounding leads (NEW entries first) because it is the premise everything else
stands on; a human skims what he is shown first. Then ask an interactive question:
"Approve this contract?" with options **Approve (Recommended)**,
**Request changes**, and **Abort**. Do not proceed to Phase 4 until the
answer is Approve. On Request changes, route the human's feedback through the
same channel as the amendment rule above: the generator proposes the
amendment with a reason, the evaluator must accept, the issue comment gets
updated with an "Amended" note, then this gate re-runs on the result. On
Abort, stop the run without building.

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

## Phase 4: Execution (labor-tier team, straight from the contract)

There is no technical planning phase and no PLAN.md: the contract is the
plan, and technical decisions belong to the builders, made against the code
as they work. The `grounding` block is the one exception: which existing
system a criterion reads or extends is a verified fact, fixed in Phase 2, and
not a builder's to re-decide. As lead, do the split inline before spawning
anyone:

- Group the contract criteria into workstreams (often just one for
  issue-sized work) and give each workstream a file territory, so parallel
  teammates never edit the same files.
- Sequence dependent workstreams; run independent ones in parallel.
- Spawn one labor-tier teammate per workstream with: the issue context, the
  contract criteria its work must satisfy, its file territory, and the
  conventions to follow.
- Prefer the project's specialized agent types when they fit the stack.
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
  No issue numbers, no contract clause ids, no `harness/` paths, no role
  names ("the evaluator", "eval seam", "the contract") in comments,
  docstrings, test names, section headers, or user-facing strings. You are
  handing each teammate a brief full of `C13` and `#62`, and a builder that
  reads `C13` in its brief will write `# C13:` in the code unless told not
  to. Say it explicitly, every time. See Guardrails.

Cross-workstream integration defects are caught by the behavioral
evaluation (Phase 6), not here.
If the split reveals the request is ambiguous, ask before building.

## Phase 5: Static verification

Run the project's checks where they exist: build, tests, linter, formatter,
type checks. Report real output; do not claim success without evidence. Fix
failures within scope before proceeding. The app must build and start before
Phase 6 has anything to test.

## Phase 6: Behavioral evaluation loop (the core)

Loop, max `--max-rounds` (default 5) rounds:

1. Start the running artifact the way the surface prescribes: run the dev
   server, install the package into a clean environment, build the
   executable, boot the service, or launch the simulator.
2. Spawn the **evaluator** (judgment-tier) with the tools its surface needs: a browser
   automation driver for `web`, direct UI control for `native`, Bash for
   `library`, `cli`, and `service`. It receives ONLY: the agreed contract, how to reach the running
   artifact, and the issue body.

   Its role prompt, keeping the bracketed clause that matches the surface:

   > You are a harsh, skeptical QA engineer. The builder's claims mean
   > nothing; only what you observe counts. You have not read the
   > implementation or the builder's tests, and you will not: everything you
   > need is in the contract. For EACH contract criterion: exercise it
   > against the running artifact like a demanding user.
   >
   > [web, native: click, type, use the keyboard, resize, try to break it.
   > Check the console and network for errors while you do.]
   >
   > [library, cli, service: write throwaway scripts in harness/eval/ that
   > drive the public surface exactly as a consumer would. Feed it empty,
   > malformed, and boundary input. Assert on return values, exception types
   > and messages, exit codes, stdout, and stderr.]
   >
   > Mark each criterion pass or fail in harness/contract.json. For every
   > failure write into harness/critique.md: the criterion id, exact
   > reproduction steps, what you observed, what the contract required.
   > FORBIDDEN verdicts: "mostly works", "minor, acceptable", "fix later". A
   > criterion passes or it fails. End with PASS (all criteria pass) or
   > ROUND_FAILED.

3. On PASS: exit the loop. Track pass/fail counts per issue across rounds;
   the evaluator judges the integrated app, but results are reported per
   issue.
4. On ROUND_FAILED: spawn a fresh labor-tier fixer with the critique and the
   contract (not the evaluator's transcript). It fixes exactly what the
   critique raises, appends to `harness/progress.json`, and you re-run
   static checks, then loop.
5. **Restart rule:** if the same criterion fails 3 consecutive rounds,
   patching is not converging. Instruct a fresh generator to DELETE that
   feature's implementation and rebuild it from scratch against the
   criterion. This counts as one round.
6. If max rounds are exhausted without PASS: stop, report which criteria
   still fail and why, and let the user decide. Do not ship a failing
   contract silently, and do not weaken criteria to make them pass (see
   amendment rule).

## Phase 7: Pull request

1. Stage and commit with a clear, conventional message following the repo's
   conventions. `harness/` stays out of the commit.
2. Push the worktree branch. Open the PR with `gh pr create --base "$BASE"`.
   The body must include:
   - Summary of what changed and why.
   - The workstream split and any deviations.
   - **Contract results per issue: criteria passed/total for each issue and
     for integration, evaluation rounds used, any amendments made and
     why.**
   - Static verification results with real output.
   - `Closes #<n>` for each issue this resolves.
3. Post the final contract state (passes, rounds, amendments) as a comment
   on each issue, its own criteria only.
4. Return the PR URL to the user.

## Guardrails

- The evaluator never reads generator transcripts, and no agent other than
  the generator-evaluator pair (via the amendment rule) may modify agreed
  contract criteria. You do not weaken a criterion to make a round pass.
- **Never relay a contract whose grounding you have not reproduced.** An EXISTS
  entry is a claim about the world; you check it with a command, not with
  trust. A criterion can be re-attacked in a later round, but a false premise
  is silently inherited by every criterion written on top of it, and by every
  sibling issue that builds on the same substrate afterwards. A fact does not
  go stale the way a design does; it becomes false and fails its grep.
- **The evaluator never runs the builder's test suite and never reads the
  implementation.** Builder-written tests encode the builder's understanding,
  so a green suite certifies whatever misunderstanding produced the bug. That
  suite runs in Phase 5 as static verification; it is not evaluation. The
  evaluator writes its own scripts, from the contract alone.
- The evaluator's scripts stay in `harness/eval/`, uncommitted, and are never
  merged into the project's test suite.
- **This run's nomenclature must not survive this run.** `harness/` is
  gitignored and thrown away. The contract lives only in an issue comment,
  which is not in the repository. Bare clause ids collide across contracts:
  the `C7` of one issue is not the `C7` of the next. So a comment citing
  `C13`, `#62`, `harness/eval-notes.md`, or `contract §3.3` is unresolvable
  to a reader holding only a clone. It is a dead end carrying a citation's
  confidence, and it rots the moment the cited artifact is deleted or the
  numbering is reused. Role names are worse than useless: they misdescribe
  the code. A production guard whose docstring calls it an "eval seam" reads
  as scaffolding, and scaffolding invites deletion. Comments state the
  substance, in terms a stranger can check against the code in front of them.
  Git history and the issue thread answer "which change introduced this and
  why was it argued that way"; the comment answers "what constraint does this
  code satisfy". This binds you as lead too, when you relay a critique or
  write a commit body's neighbouring code.
- Stay within the scope set by the contract. Adjacent work becomes
  a follow-up note, not PR growth.
- Never commit or push to the base branch directly; everything reaches it
  via the PR.
- Report outcomes faithfully: failing criteria, skipped checks, and
  exhausted rounds are surfaced, never buried.
- Do not delete or overwrite files you did not create without surfacing the
  conflict first.
