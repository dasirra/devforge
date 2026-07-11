# Task 1 findings: Claude Code plugin skill naming and discovery

Sources fetched and verified live on 2026-07-11:
- https://code.claude.com/docs/en/skills.md
- https://agentskills.io/specification

## Resolved values

```
SKILL_NAME_PATTERN=<workflow>        # e.g. name: building, name: planning, name: interview
SKILL_DIR_PATTERN=skills/<workflow>/ # e.g. skills/building/, skills/planning/, skills/interview/
AUTO_DISCOVERED=yes
```

**`SKILL_NAME_PATTERN` does NOT default to `forge-<workflow>`. It is `skills/<workflow>/`, i.e. no `forge-` prefix on the directory.** This did NOT fall back to the brief's stated default (`forge-<workflow>` dir / `<workflow>` name). The docs directly answer the question and the directory part of that default is wrong. See "Discrepancy" below.

## Evidence

### 1. Auto-discovery (brief Q1): CONFIRMED, no fallback needed

The Claude Code skills doc's "Where skills live" table lists:

> | Plugin | `<plugin>/skills/<skill-name>/SKILL.md` | Where plugin is enabled |

alongside Enterprise/Personal/Project rows, with no mention of a manifest entry required for plugin skills. `plugin.json` needs no `skills` field; presence of the `skills/` directory in the plugin is sufficient. → `AUTO_DISCOVERED=yes`.

### 2. What `name:`/directory combination yields `/forge:building` (brief Q2): CONFIRMED, differs from brief's default

The doc's "How a skill gets its command name" table states, exactly:

> | Plugin `skills/` subdirectory | Directory name, namespaced by plugin | `my-plugin/skills/review/SKILL.md` → `/my-plugin:review` |

and separately:

> The frontmatter `name` field sets the display label shown in skill listings and, except for a plugin-root `SKILL.md`, does not change what you type after `/`.

Applied to this repo: the plugin is named `forge` (per `.claude-plugin/plugin.json`, `"name": "forge"`). For a non-root plugin skill, the **directory name itself becomes the part after the colon**. Claude Code prepends the plugin's own name as the namespace automatically. It does not concatenate a `forge-` prefix already baked into the directory name.

So to get `/forge:building`:
- Directory must be `skills/building/` (bare workflow name), **not** `skills/forge-building/`.
- If the directory were `skills/forge-building/`, the resulting invocation would be `/forge:forge-building`, not `/forge:building`.
- The `name:` frontmatter value is not what drives the invocation here (only plugin-root `SKILL.md` uses `name:` for that); but agentskills.io requires `name:` to equal the directory name anyway (see below), so `name: building` in `skills/building/SKILL.md` is both spec-compliant and consistent with the intended `/forge:building` invocation.

### 3. agentskills.io `name` field constraints (brief Q3): CONFIRMED

From the specification page's frontmatter table and `name` field section:

> `name`: Required. Max 64 characters. Lowercase letters, numbers, and hyphens only. Must not start or end with a hyphen.
>
> The required `name` field:
> * Must be 1-64 characters
> * May only contain unicode lowercase alphanumeric characters (`a-z`, `0-9`) and hyphens (`-`)
> * Must not start or end with a hyphen (`-`)
> * Must not contain consecutive hyphens (`--`)
> * **Must match the parent directory name**

So a colon (`:`) is **not permitted** in `name:`. `name: forge:building` would be spec-invalid. And the only two required fields are `name` and `description`; everything else (`license`, `compatibility`, `metadata`, `allowed-tools`) is optional.

## Discrepancy flagged for Tasks 2-9

`docs/plans/2026-07-11-forge-skills-conversion.md` (Tasks 2-9, already committed) hardcodes `skills/forge-<workflow>/SKILL.md` throughout (e.g. `skills/forge-interview/`, `skills/forge-building/`). Based on the evidence above, that directory convention produces `/forge:forge-interview`, `/forge:forge-building`, etc., not the intended `/forge:interview`, `/forge:building`. The correct directory convention per current Claude Code docs is the bare workflow name: `skills/interview/`, `skills/planning/`, `skills/building/`, with `name: interview` / `name: planning` / `name: building` in each `SKILL.md` (matching the directory, per agentskills.io).

This finding is recorded here per Task 1's scope (research + decision only). Whoever executes Tasks 2-9 should use `SKILL_DIR_PATTERN=skills/<workflow>/` (no `forge-` prefix) instead of the plan's literal `skills/forge-<workflow>/` paths, and update the plan document's Task 2-9 file paths accordingly before or during execution.

## Summary

```
SKILL_NAME_PATTERN=<workflow>          # name: building | name: planning | name: interview
SKILL_DIR_PATTERN=skills/<workflow>/   # skills/building/ | skills/planning/ | skills/interview/
AUTO_DISCOVERED=yes
```

Fallback to brief's stated default: NOT used. Docs answered all three questions directly; the brief's default directory convention (`forge-<workflow>`) was checked against the docs and found incorrect, so the docs-derived value is recorded instead.
