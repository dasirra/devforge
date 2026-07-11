---
name: planning
description: Adversarial PM-level planning. A planner drafts an epic with user stories, a critic attacks it, they iterate, then everything lands on GitHub immediately as issues for async human review. Technical contracts are deliberately excluded; those get negotiated at /forge:building time.
disable-model-invocation: true
---

# /forge:planning

> **Harness binding.** This command is written in harness-neutral terms: role
> tiers (e.g. `judgment-tier`, `labor-tier`) and generic verbs, instead of one
> agent's tool and model names. Before acting, load the binding for your
> environment from [`docs/harness-bindings/`](../../docs/harness-bindings/README.md)
> and resolve every neutral term to the concrete tool or model it names; when
> the prose spawns a subagent of a given tier, use the model the binding maps
> that tier to.

You orchestrate an adversarial planning session and file the result as a GitHub
Epic with PM-level sub-issues, created for asynchronous human
review. You do NOT write code, you do NOT plan technical implementation, and
you do NOT run /forge:building. Two subagents with separate context windows do the
thinking: a PLANNER drafts, a CRITIC attacks. You relay artifacts between them
and file the outcome.

Requested input:
$ARGUMENTS

If `$ARGUMENTS` is empty or was not substituted, treat the user's request itself as the arguments.

## Operating principles

- **PM altitude only.** Issues contain user stories, goals, human-readable
  acceptance criteria, proposed behavior, scope boundaries, dependencies, and a
  Grounding block. NEVER schemas, libraries, or architecture, and no file paths
  or function names outside Grounding. Granular testable contracts are negotiated
  adversarially at /forge:building time, against the codebase as it exists then.
  Freezing them now would let them go stale. Grounding is different: it records
  what already exists, and a fact does not go stale, it becomes false and fails
  its check loudly.
- **Issues must be self-sufficient for review.** A human reading only the
  GitHub issues must be able to understand the full proposed solution: what
  will be built, how it behaves, and why. Use prose plus mermaid diagrams of
  user flows, states, and interactions.
- **Separate contexts are the mechanism.** The critic only ever sees the plan
  artifact, never the planner's reasoning. Do not summarize one agent's
  thinking to the other; pass files.
- **Bounded adversarial rounds.** Max 3 critique cycles. Unresolved
  disagreements are not discarded and not resolved by you; they are embedded
  in the affected issues for the human to arbitrate on GitHub.
- **Create-first, review-async.** Everything is created immediately. The
  human reviews the issues on GitHub, edits them freely, and arbitrates
  contested items before implementing.
- **One issue = one independently-PR-able unit of work.** Not a whole phase,
  not a single function.
- **Right-size it.** If the feature genuinely decomposes to one or two issues,
  skip the epic and just create the issues.

## Phase 0: Interpret the input

Decide which shape $ARGUMENTS is:

1. **Empty** -> the feature is defined by the current conversation. Synthesize
   it from what has been discussed (designs, decisions, constraints agreed).
2. **A path to a .md file** -> read it; that spec is the feature.
3. **Free-form text** -> that text is the feature prompt.

Restate the feature as a short problem statement plus a bullet list of the
capabilities you extracted. If it is ambiguous or clearly spans several
unrelated features, ask the user to narrow or split it before continuing.

## Phase 1: Preconditions

Verify and, if needed, offer to fix:

- Inside a git repository with a GitHub remote (`gh repo view`). If not, stop
  and report.
- `gh` is authenticated (`gh auth status`).
- An `epic` label exists (`gh label list`). If missing, create it:
  `gh label create epic --description "Tracking issue spanning multiple child issues" --color 5319e7`
Read the repo's existing conventions so you mirror them:

- `gh issue list --state all --limit 20 --json number,title,labels` for title
  prefixes and label usage.
- If an existing epic is present, read its body
  (`gh issue view <n> --json body`) and copy its structure.

## Phase 2: Grounding and reality check (bounded; skip only if greenfield)

Dispatch ONE exploratory sub-agent with a narrow question: what does this product
currently do in the area the feature touches, and what adjacent capabilities
already exist? This serves the critic's feasibility judgment, sane phase
ordering, and grounding. If the spec carries a Grounding section, verify each
entry still holds against the repo; either way, produce a grounding table:
every capability, store, or system the plan treats as already existing,
confirmed with a file or symbol reference, or explicitly marked NEW. Grounding
facts, including the reference that proves them, are the one kind of technical
content permitted into issue bodies, quoted in each child's Grounding block. No
other technical detail may leak. For a greenfield project, skip this phase.

## Phase 3: Planner drafts (subagent, fresh context)

Spawn a subagent (model: judgment-tier) with this role:

> You are a product planner. Given this feature statement [and product
> context], write plan.md containing:
>
> - Product goal: what and why, one short paragraph.
> - Proposed solution, at product level: how the feature works from the
>   user's perspective, and the key product decisions with a one-line why
>   for each (e.g. "single shared list, not per-user lists: keeps onboarding
>   at zero steps").
> - Mermaid diagrams wherever they aid understanding: user flows
>   (`flowchart`), screen/entity states (`stateDiagram-v2`), or user-system
>   interactions (`sequenceDiagram` at the level of "user does X, product
>   responds Y"). Every non-trivial issue should let a reviewer understand
>   the intended behavior from the diagram alone.
> - 3-8 sub-issues, each with: a title, a user story ("As a <user>, I want
>   <capability>, so that <outcome>"), a proposed-behavior description
>   covering empty/error/edge states, human-readable acceptance criteria
>   (observable behavior, no implementation), and out-of-scope notes.
> - Ordered phases, where each phase leaves something shippable and
>   demonstrable. Record blocked-by / blocks between issues.
>
> Rules: no technical details of any kind. You define what the product should
> do and in what order, nothing about how. If the feature genuinely needs only
> 1-2 issues, say so; do not manufacture an epic.

Write plan.md to a working scratch location.

## Phase 4: Critic attacks (subagent, fresh context, adversarial)

Spawn a separate subagent (model: judgment-tier) that receives ONLY plan.md and the
original feature statement, with this role:

> You are a skeptical product reviewer. Your job is to find what is wrong with
> this plan, not to improve its wording. Attack, in order of severity:
>
> - Missing scope: user needs or edge cases the stories silently drop.
> - Unverifiable acceptance criteria: anything a reviewer could not check by
>   using the product ("works well", "is intuitive").
> - Diagram/story mismatch: flows that skip error paths, states with no way
>   out, diagrams that contradict the acceptance criteria.
> - Phase ordering: phases that are not shippable alone, dependencies that are
>   wrong or missing, hidden coupling between issues.
> - Scope creep: stories that do not serve the stated goal, gold-plating.
> - Wrong granularity: issues too big to be one PR-able unit, or too small to
>   matter.
>
> Write critique.md: one finding per line with severity (BLOCKER / MAJOR /
> MINOR) and the exact story, criterion, or diagram it targets. If the plan is
> genuinely sound, say APPROVED and stop. Do not invent objections to seem
> rigorous.

## Phase 5: Iterate (max 3 rounds)

Hand critique.md to a FRESH planner instance (new subagent, not the original;
a planner defending its own draft in continued context is the self-criticism
trap this design exists to avoid) along with plan.md. It revises the plan,
addressing or explicitly rejecting each finding. A rejection needs one line of
reasoning, recorded in plan.md under a "Contested" section. Re-run the critic
on the revision. Stop when the critic says APPROVED, or after 3 rounds. Never
resolve a disagreement yourself; contested items travel into the issues.

## Phase 6: Create on GitHub (immediately)

Create the issues and wire the relationships. Reliable sequence:

1. **Create the epic** with
   `gh issue create --label epic --label enhancement`.
   Capture its number.
2. **Create each child** with
   `gh issue create --label enhancement` (plus any component
   label the repo already uses). Capture each child's issue number AND its
   database id: `gh api repos/{owner}/{repo}/issues/<n> --jq '.id'`.
3. **Resolve cross-references.** Child and epic bodies refer to sibling issues
   by number, which are unknown until creation. Author bodies with stable
   placeholder tokens (for example `__C3__`, `__EPIC__`), then after all
   numbers are known do a second pass substituting real numbers and update
   each body with `gh issue edit <n> --body-file <resolved>`. This avoids
   forward-reference problems.
4. **Link native sub-issues.** For each child, call the REST API with the
   child's DATABASE id (not its issue number):
   `gh api -X POST repos/{owner}/{repo}/issues/<EPIC>/sub_issues -F sub_issue_id=<child_db_id>`
   This makes children render in the epic's Sub-issues panel with progress
   tracking.
5. **Verify:** `gh api repos/{owner}/{repo}/issues/<EPIC>/sub_issues` lists
   all children, and no `__token__` placeholders remain in any body.

Scripting note: the default macOS bash is 3.2 and lacks associative arrays
(`declare -A`). Use indexed/parallel arrays, or just issue the gh calls
directly.

### Epic body template

````
> Adversarially planned, pending human review. Review each sub-issue and
> edit as needed before implementing.

## Goal
<what and why, short paragraph>

## Proposed solution
<how the product will work once built, from the user's perspective;
enough that a reviewer understands the whole shape without opening
any child issue>

```mermaid
flowchart TD
  <high-level user journey through the feature>
```

## Key decisions
| Decision | Choice | Why |
|---|---|---|
| ... | ... | ... |

## Phased delivery (build order)
**Phase 1 - <name>**
- [ ] #<a> - <title> (needs #<x>)
...

```mermaid
flowchart LR
  <dependency graph by issue number>
```

## Out of scope
- ...

## Planning notes
Adversarially reviewed: <n> critique rounds, <n> findings addressed.
Contested items pending human arbitration: <links to affected issues, or "none">.
Technical contracts: negotiated at /forge:building time, per issue.

## References
- <spec path, conversation summary, or source documents this plan came from>
````

### Child body template

````
> Pending human review. Edit freely before implementing.

**Phase <n> · Part of #<EPIC>**

## User story
As a <user>, I want <capability>, so that <outcome>.

## Proposed behavior
<what the user sees and does, including empty/error/edge states, in prose>

```mermaid
<the diagram that best explains this issue: user flow, state diagram,
or user-system sequence. Include unhappy paths. Omit ONLY if the issue
is trivially linear.>
```

## Acceptance criteria
- <observable behavior, human-readable>
...

## Out of scope
- ...

## Grounding
<facts only, from the planning grounding table: the existing systems this
issue reads or extends, by name and location, and anything it must CREATE
because it does not exist, marked NEW. Omit only if the issue touches
nothing that outlives its own PR.>

## Dependencies
**Blocked by** #<...>. **Blocks** #<...>.

## Contested (human arbitration needed)
<only if unresolved planner/critic disagreements touch this issue:
"CRITIC: <objection> / PLANNER: <rejection reasoning>". Omit if none.>

---
*Granular contract to be negotiated by generator/evaluator at /forge:building time
and posted back to this issue as a comment.*
````

## Phase 7: Report

Print for the user:

- The epic URL and a table of every child: phase, title, one-line user story,
  blocked-by, and whether it carries contested items.
- The critique trail: what the critic caught each round and what changed.
- Contested items needing arbitration, with links.
- The runbook:

```
Review the issues on GitHub, arbitrate contested items, and edit freely
before implementing.

Then implement one phase per /forge:building run, review/merge between phases:

  Phase 1:  /forge:building #<a> #<b>
            (review the PR, run checks, merge)
  Phase 2:  /forge:building #<c> #<d>
  ...

Critical path: #<...> -> #<...> -> #<...>
Startable first (no deps): #<...>
```

## Guardrails

- No technical decisions in any issue body. The Grounding block is exempt: it
  carries verified facts about existing systems, with their locations. If the
  planner leaks a decision, strip it before filing; if it leaks a fact, move it
  into Grounding instead of deleting it.
- Diagrams and solution descriptions stay at product level: user flows,
  states, interactions. No component diagrams, no data models, no API
  shapes. If a diagram names a file, table, or endpoint, redraw it.
- Do not resolve planner/critic disagreements yourself; embed them and let
  the human arbitrate on GitHub.
- Do not run /forge:building. Your output is the issues plus the runbook.
- Mirror the repo's existing conventions; do not invent labels or prefixes
  beyond `epic`.
- Convert any relative dates in text to absolute dates.
