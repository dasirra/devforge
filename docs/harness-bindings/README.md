# Harness bindings

Forge's skills (`skills/*/SKILL.md`) are written in harness-neutral prose: role
tiers (`judgment-tier`, `labor-tier`) and generic verbs, instead of any one
agent's tool and model names. Each supported harness has a binding doc in this
directory that maps those neutral terms to its concrete tools and models. A
skill's **Harness binding** preamble points the running agent at the binding
for its environment; the agent resolves every neutral term through it before
acting.

Each binding is **self-contained**: it describes only its own harness and does
not reference the others. To learn what a harness does, read its binding; to
compare harnesses, read this file.

## Available bindings

- [`claude-code.md`](claude-code.md): Claude Code. Judgment/labor tiers map to
  distinct models (opus / sonnet); binding is an exact-string substitution
  (a round-trip check verifies it reconstructs the original skill text).
- [`pi.md`](pi.md): Pi with open-weight models. Every tier maps to one model
  (GLM-5.2); binding is semantic (the agent interprets each phrase). Requires
  forked-context subagents (`pi-subagents`); the `native` evaluation surface is
  unavailable.

## Why single-model is fine

The adversarial pairs (planner vs critic, generator vs evaluator) do not depend
on the two sides running different models. What makes a subagent an adversary
is that its context is separated from its counterpart's, not its weights. So a
harness that runs one model for every role loses the cost split but not the
method. The consequence to respect: with one model on both sides, context
separation is the only thing preventing a rubber-stamp, so a harness without
forked-context subagents cannot run forge honestly.

## Adding a harness

Copy the neutral-phrase list from any existing binding, map each phrase to your
harness's equivalent, and note the harness-specific caveats (whether it has
forked-context subagents, which evaluation surfaces it supports, whether it
selects models per role). Keep it self-contained.
