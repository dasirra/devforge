# Pi binding

Forge's skills (`skills/interview/SKILL.md`, `skills/planning/SKILL.md`,
`skills/building/SKILL.md`) are written in harness-neutral prose: instead of naming
a specific coding agent's tools or models, they use neutral verbs and tier
markers that each harness binds to its own equivalents. This doc is the Pi
binding: it maps every neutral phrase used in the prose to the Pi coding
agent's concrete tool or, for tiers, the open-weight model that realizes it.

| Neutral phrase (in skills/) | Pi binding |
| --- | --- |
| judgment-tier | GLM-5.2 |
| labor-tier | GLM-5.2 |
| an interactive question | a plain chat question to the user, options listed with the recommended one first |
| the worktree helper | `git worktree add` |
| exploratory | a read-only subagent (the `subagent` tool from `pi-subagents`, forked context) |
| a working scratch location | a `harness/` directory inside the worktree, git-excluded (per the building skill's `.git/info/exclude` step) |

Read each neutral phrase in the prose as the Pi tool, model, or behaviour its
right-hand value describes, and realise that behaviour: this is a semantic
binding the Pi agent interprets, not a literal find-and-replace.
Several right-hand values are descriptions rather than drop-in tokens (for
example `exploratory` and `an interactive question`): when the prose says "an
exploratory subagent", spawn a read-only forked-context subagent; do not paste
the description into the sentence.

## Single model, both tiers

Pi runs GLM-5.2 for every role; both `judgment-tier` and `labor-tier` map to
it. The judgment/labor distinction stays legible in the prose but does not
drive model selection here; there is no separate tier to select. What makes a
subagent an adversary is its separated context, not its weights, so a single
model does not weaken the design on its own.

There is one consequence to respect. With the same model on both sides of every
pair, **context separation is the only thing preventing a rubber-stamp**. The
`exploratory`/subagent binding below is therefore load-bearing. Before trusting
a run, confirm the evaluator genuinely pushes back rather than passing every
criterion on the first round.

## Required: forked-context subagents

Forge's adversaries (planner vs critic, generator vs evaluator) only work
because each cannot see the other's reasoning. Pi core ships no subagent tool;
the `pi-subagents` package provides one with forked contexts. **Do not run
forge on a Pi without it.** Without forked contexts the pipeline collapses into
self-review, the exact failure the design exists to prevent, and no amount of
prose fixes that. Do not simulate a subagent by continuing in the same context;
if the package is absent, stop and say so rather than faking the separation.

## Evaluation surfaces (not yet neutralized)

The `web` / `native` / browser-MCP lines in `skills/building/SKILL.md` were not
neutralized (that was out of scope for the neutralization), so they
still name tools that may not exist on Pi. Read them as follows until that
follow-up lands:

- `native` (computer-use) surface: **unavailable**. Skip any step whose only
  observation path is native, tell the user, and continue.
- `web`: drive the browser with **Playwright via Bash**, not a browser MCP.
- `library` / `cli` / `service`: need only Bash; these port unchanged.
