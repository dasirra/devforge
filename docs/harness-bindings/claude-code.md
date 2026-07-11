# Claude Code binding

Forge's skills (`skills/interview/SKILL.md`, `skills/planning/SKILL.md`,
`skills/building/SKILL.md`) are written in harness-neutral prose: instead of
naming a specific coding agent's tools or models, they use neutral verbs and
tier markers that each harness binds to its own equivalents. This doc is
the Claude Code binding: it maps every neutral phrase used in the prose to
the concrete Claude Code tool or model tier that realizes it.

| Neutral phrase (in skills/) | Claude Code binding |
| --- | --- |
| judgment-tier | opus |
| labor-tier | sonnet |
| an interactive question | AskUserQuestion |
| the worktree helper | EnterWorktree |
| exploratory | Explore |
| a working scratch location | the session scratchpad directory |

Every left-hand phrase is the exact string as it appears in the skills;
binding is a literal substitution of that phrase by its right-hand value.
The `exploratory` row is a single-word swap: it marks Claude Code's
read-only **Explore** subagent, and substitution replaces only the word
`exploratory` with `Explore`, leaving the following generic `subagent` /
`sub-agent` in the prose unchanged.
