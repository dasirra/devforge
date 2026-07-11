# Claude Code binding

Forge's commands (`commands/interview.md`, `commands/planning.md`,
`commands/building.md`) are written in harness-neutral prose: instead of
naming a specific coding agent's tools or models, they use neutral verbs and
tier markers that any harness can bind to its own equivalents. This doc is
the Claude Code binding: it maps every neutral phrase used in the prose to
the concrete Claude Code tool or model tier that realizes it. A planned
sibling Pi binding (issue #5) will map the same neutral verbs to open-model
equivalents, so the same commands can run unmodified on either harness.

| Neutral phrase (in commands/) | Claude Code binding |
| --- | --- |
| judgment-tier | opus |
| labor-tier | sonnet |
| an interactive question | AskUserQuestion |
| the worktree helper | EnterWorktree |
| exploratory | Explore |
| a working scratch location | the session scratchpad directory |

Every left-hand phrase is the exact string as it appears in `commands/`;
binding is a literal substitution of that phrase by its right-hand value.
The `exploratory` row is a single-word swap: it marks Claude Code's
read-only **Explore** subagent, and substitution replaces only the word
`exploratory` with `Explore`, leaving the following generic `subagent` /
`sub-agent` in the prose unchanged.
