# Design: convert forge commands to skills (cross-platform)

Date: 2026-07-11
Status: approved design, pending spec review

## Problem

Forge ships as a Claude Code plugin whose three workflows are **commands**
(`commands/interview.md`, `commands/planning.md`, `commands/building.md`),
invoked as `/forge:building #3`. Commands are a Claude Code construct. Pi, and
the Agent Skills ecosystem generally, has no commands; its native unit is the
**skill**. To run forge on Pi we were about to bridge command-format to
skill-format, which reintroduces the two-copies-drift problem the harness
neutralization work spent its whole effort eliminating.

## Key enabling fact

Claude Code has merged skills and commands. A skill can be invoked explicitly
as `/forge:building #3`, supports the `$ARGUMENTS` placeholder exactly like a
command, and with `disable-model-invocation: true` is explicit-only (the user
invokes it; Claude never auto-fires it). So on Claude Code a skill is a strict
**superset** of a command: converting loses nothing. On Pi the skill is the
native unit. Therefore the skill is the single representation that runs on both
platforms with no bridge.

## Goal

Convert the three commands into skills so that one source tree runs natively on
Claude Code and Pi, maximizing compatibility across agent platforms, with no
regression to the current Claude Code invocation (`/forge:building #3` still
works, still explicit, still staged).

## Design

### Structure

Replace `commands/` with a `skills/` tree, one skill per workflow, each holding
the existing neutralized body verbatim:

```
skills/
  interview/SKILL.md
  planning/SKILL.md
  building/SKILL.md
docs/harness-bindings/          # unchanged: claude-code.md, pi.md, README.md
package.json                    # new: Pi manifest
.claude-plugin/plugin.json      # updated: wording only
.claude-plugin/marketplace.json # updated: wording only
```

One representation, two consumers: Claude Code auto-discovers `skills/` from the
plugin; Pi discovers the same directory via `package.json`.

### Frontmatter

Each `SKILL.md` replaces command frontmatter with skill frontmatter:

```yaml
---
name: forge:building
description: <the current one-line description, verbatim>
disable-model-invocation: true
---
```

- `name` carries the `forge:` namespace so the Claude Code invocation stays
  `/forge:building`. (Exact namespacing for a plugin skill is an open item, see
  below.)
- `description` is copied unchanged from the current command frontmatter.
- `disable-model-invocation: true` makes the skill explicit-only on Claude Code,
  which is required for a human-gated staged pipeline (you do not want Claude
  auto-running `/forge:building` because code "looks ready").
- `argument-hint` is dropped; it is not a skill frontmatter field.

### Arguments

`$ARGUMENTS` stays in each body where the current commands use it; Claude Code
substitutes it. Because a harness may not substitute `$ARGUMENTS`, each body
gets one robustness line near the top of its interpret-the-request step:

> If arguments were not substituted into `$ARGUMENTS`, take the issue
> references and flags from the user's request instead.

This makes argument handling correct on Claude Code (substitution) and on any
harness that instead passes the request as context (extraction).

### Manifests

- **Claude Code**: keep `.claude-plugin/plugin.json` and `marketplace.json`.
  Skills ship inside the plugin and are auto-discovered; `/plugin install`
  continues to work. Only descriptive wording changes ("commands" -> "skills").
- **Pi**: add `package.json`:
  ```json
  {
    "name": "forge",
    "version": "<current>",
    "description": "<current>",
    "keywords": ["pi-package", "workflow", "planning", "github"],
    "pi": { "skills": ["./skills"] }
  }
  ```
  No `.pi/extensions/*.ts` in this round. The skill body's Harness-binding
  preamble already directs the agent to read `docs/harness-bindings/pi.md`, so
  binding resolution needs no extension.

### What does not change

The neutralized bodies, the harness-agnostic binding preamble, the binding
layer (`docs/harness-bindings/*`), and the `gh` usage are all untouched. This
change is purely the packaging unit (command -> skill). The harness-neutrality
work sits underneath it, unchanged.

## Invocation model

| Platform | Invocation | Arguments | Auto-trigger |
| --- | --- | --- | --- |
| Claude Code | `/forge:building #3` | `$ARGUMENTS` substituted | disabled via `disable-model-invocation` |
| Pi | `/skill:forge:building` (or by request) | from request; `$ARGUMENTS` if supported | per Pi's skill model (verify) |

## Open items to verify during implementation (do not assume)

1. **Pi argument passing.** Does Pi substitute or otherwise pass arguments to an
   invoked skill? If not, the extraction fallback covers it, but confirm.
2. **`disable-model-invocation` on Pi.** Does Pi read, ignore, or error on this
   frontmatter field? It must at least not break skill loading.
3. **Claude Code plugin skill discovery and naming.** Confirm plugin skills in
   `skills/` are auto-discovered, and confirm the exact invocation name for a
   namespaced plugin skill (`forge:building` vs `forge-building` vs
   `plugin-name:skill`). Adjust `name` and directory names to whatever yields
   `/forge:building`.
4. **agentskills.io spec constraints.** Check `agentskills.io/specification` for
   any constraint on frontmatter fields or skill directory naming that affects
   the above.

## Out of scope (YAGNI)

- A `.pi/extensions/forge.ts` that auto-injects the binding into context. The
  preamble covers binding-loading without it; revisit only if the manual read
  proves unreliable on Pi.
- Any change to the pipeline behavior, the bindings, or the neutralized prose.

## Testing / verification

- **Claude Code (no regression):** after conversion, invoke each of the three
  skills with `/forge:<name>` and confirm identical behavior to the current
  commands (explicit invocation, `$ARGUMENTS` received, no auto-trigger). A
  round-trip is not applicable here (this is a packaging change, not a prose
  edit); the check is behavioral parity of invocation.
- **Pi:** install the package on a Pi environment, invoke each skill, confirm it
  loads, reads `pi.md`, resolves the neutral verbs, and runs. This is the same
  Pi validation already done manually for the command form; repeat it for the
  skill form.
- **Structural checks:** every `SKILL.md` has valid `name` + `description`
  frontmatter; `package.json` `pi.skills` points at the real directory; no
  `commands/` remains; `plugin.json`/`marketplace.json`/README no longer say
  "commands"; the three neutralized bodies are byte-identical to their
  `commands/*.md` source except for the frontmatter block and the one
  argument-fallback line.

## Risks

- **Namespacing/invocation-name drift.** If a plugin skill cannot reproduce the
  `/forge:building` name exactly, existing Claude Code users' muscle memory
  changes. Mitigation: verify item 3 before finalizing names; if `forge:` is not
  achievable, decide the closest acceptable name explicitly.
- **Pi arg model weaker than expected.** If Pi neither substitutes `$ARGUMENTS`
  nor reliably lets the model extract flags, complex invocations
  (`--no-gate --max-rounds 3`) may be fragile on Pi. Mitigation: the extraction
  fallback plus explicit instruction; document the limitation if it persists.
- **Marketplace listing.** Converting may affect the Claude Code marketplace
  entry. Mitigation: keep `.claude-plugin/*`; verify `/plugin install` still
  resolves the plugin with skills.
