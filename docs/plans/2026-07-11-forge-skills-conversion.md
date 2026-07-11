# Forge Skills Conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert forge's three Claude Code commands into skills so one source tree runs natively on both Claude Code and Pi.

**Architecture:** Each `commands/<name>.md` becomes `skills/<name>/SKILL.md` with skill frontmatter (`name`, `description`, `disable-model-invocation`). The neutralized body is preserved verbatim plus one argument-fallback line. Claude Code keeps its plugin manifest; Pi gets a `package.json` declaring the same `skills/` directory. The binding layer (`docs/harness-bindings/*`) is untouched.

**Tech Stack:** Markdown skill files, JSON manifests, `gh`/`git`, bash verification. No application code, no test framework; "tests" are structural checks (grep/diff/JSON-parse) and, for the two behavioral tasks, live invocation.

## Global Constraints

- Never use the em dash symbol (`—`) in any file. Rewrite the sentence.
- Neutralized bodies stay byte-identical to their `commands/*.md` source except: (a) the frontmatter block is replaced, and (b) exactly one argument-fallback line is inserted.
- No forbidden Claude tokens reintroduced into skill bodies: `opus`, `sonnet`, `AskUserQuestion`, `EnterWorktree`, whole-word `Explore`, `scratchpad`, `code-review` must stay absent from `skills/`.
- Explicit-only invocation on Claude Code: every skill sets `disable-model-invocation: true`.
- Single source: after conversion no `commands/` directory remains; the bodies are not duplicated anywhere.
- Base branch for this work: `feature/skills-conversion` (already checked out in this worktree, forked from `building/issue-3`).

---

### Task 1: Confirm Claude Code plugin skill naming and discovery

Resolve the naming/discovery unknowns before converting, so the skill directory names and `name` frontmatter are correct on the first try.

**Files:**
- Create: `docs/plans/task1-naming-findings.md` (records the resolved naming decision; kept)

**Interfaces:**
- Produces: `SKILL_NAME_PATTERN` (the exact `name:` value that yields `/forge:<workflow>` invocation, e.g. `forge:building` or `building`) and `SKILL_DIR_PATTERN` (resolved by Task 1: `skills/building/`, bare workflow name), consumed by Tasks 2-4.

- [ ] **Step 1: Gather the facts**

Confirm, from current Claude Code docs (use the claude-code-guide agent or `https://code.claude.com/docs/en/skills.md`) and `https://agentskills.io/specification`:
1. Are plugin skills in `skills/<name>/SKILL.md` auto-discovered (no manifest listing needed)?
2. What exact `name:` value produces the invocation `/forge:building` for a skill shipped by a plugin named `forge`? (Is it `name: building` namespaced to `forge:building`, or `name: forge:building`, or `forge-building`?)
3. Does `agentskills.io` constrain the `name` field charset (does it permit `:`), and does it require any field beyond `name` + `description`?

- [ ] **Step 2: Record the decision**

Write the two resolved values to `docs/plans/task1-naming-findings.md`:
```
SKILL_NAME_PATTERN=<resolved, e.g. building>   # yields /forge:building
SKILL_DIR_PATTERN=forge-<workflow>             # or the resolved dir convention
AUTO_DISCOVERED=<yes|no; if no, note the manifest change needed>
```
Default to assume if docs are silent: dir `forge-<workflow>`, `name: <workflow>` (plugin namespaces it), auto-discovered. Note explicitly if you had to fall back to the default.

- [ ] **Step 3: Commit the finding**

```bash
git add docs/plans/task1-naming-findings.md
git commit -m "docs: resolve Claude Code plugin skill naming for conversion"
```

---

### Task 2: Convert the interview command to a skill

**Files:**
- Create: `skills/interview/SKILL.md`
- Delete: `commands/interview.md`

**Interfaces:**
- Consumes: `SKILL_NAME_PATTERN`, `SKILL_DIR_PATTERN` from Task 1.

- [ ] **Step 1: Write the verification check (expect failure now)**

Run:
```bash
test -f skills/interview/SKILL.md && echo EXISTS || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Create the skill file with new frontmatter + preserved body**

```bash
mkdir -p skills/interview
# new frontmatter (name value per Task 1; shown with the default)
cat > skills/interview/SKILL.md <<'EOF'
---
name: interview
description: Relentless one-question-at-a-time grilling interview about an idea, then synthesis into a PM-level spec document ready to pass to /forge:planning. The human-in-the-loop alignment phase; produces no code and no issues.
disable-model-invocation: true
---
EOF
# append the body of the old command with its frontmatter stripped (first --- block only)
python3 -c "import re; s=open('commands/interview.md').read(); import sys; sys.stdout.write(re.sub(r'^---\n.*?\n---\n','',s,count=1,flags=re.S))" >> skills/interview/SKILL.md
```

- [ ] **Step 3: Insert the argument-fallback line**

Insert immediately after the `Input: $ARGUMENTS` line:
```bash
python3 - <<'PY'
p='skills/interview/SKILL.md'; s=open(p).read()
anchor='Input: $ARGUMENTS\n'
add=anchor+'\nIf `$ARGUMENTS` is empty or was not substituted, treat the user\'s request itself as the arguments.\n'
assert anchor in s, 'anchor not found'
open(p,'w').write(s.replace(anchor,add,1))
PY
```

- [ ] **Step 4: Verify frontmatter + body parity + no forbidden tokens**

```bash
head -5 skills/interview/SKILL.md   # expect name/description/disable-model-invocation
# body parity: strip frontmatter from both, diff; expect ONLY the one added fallback line
diff <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('commands/interview.md').read(),count=1,flags=re.S),end='')") \
     <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('skills/interview/SKILL.md').read(),count=1,flags=re.S),end='')")
# expected: exactly one added line (the fallback), nothing else
grep -nE '\b(opus|sonnet)\b|AskUserQuestion|EnterWorktree|scratchpad|code-review' skills/interview/SKILL.md || echo "clean"
grep -nw Explore skills/interview/SKILL.md || echo "clean"
```
Expected: frontmatter correct; diff shows only the fallback insertion; `clean` printed twice.

- [ ] **Step 5: Delete the old command and commit**

```bash
git rm commands/interview.md
git add skills/interview/SKILL.md
git commit -m "feat: convert interview command to a skill"
```

---

### Task 3: Convert the planning command to a skill

**Files:**
- Create: `skills/planning/SKILL.md`
- Delete: `commands/planning.md`

- [ ] **Step 1: Verify absence (expect failure)**

Run:
```bash
test -f skills/planning/SKILL.md && echo EXISTS || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Create the skill file**

```bash
mkdir -p skills/planning
cat > skills/planning/SKILL.md <<'EOF'
---
name: planning
description: Adversarial PM-level planning. A planner drafts an epic with user stories, a critic attacks it, they iterate, then everything lands on GitHub immediately as issues for async human review. Technical contracts are deliberately excluded; those get negotiated at /forge:building time.
disable-model-invocation: true
---
EOF
python3 -c "import re,sys; sys.stdout.write(re.sub(r'^---\n.*?\n---\n','',open('commands/planning.md').read(),count=1,flags=re.S))" >> skills/planning/SKILL.md
```

- [ ] **Step 3: Insert the argument-fallback line**

Insert immediately after the `Requested input:\n$ARGUMENTS` block:
```bash
python3 - <<'PY'
p='skills/planning/SKILL.md'; s=open(p).read()
anchor='Requested input:\n$ARGUMENTS\n'
add=anchor+'\nIf `$ARGUMENTS` is empty or was not substituted, treat the user\'s request itself as the arguments.\n'
assert anchor in s, 'anchor not found'
open(p,'w').write(s.replace(anchor,add,1))
PY
```

- [ ] **Step 4: Verify**

```bash
head -5 skills/planning/SKILL.md
diff <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('commands/planning.md').read(),count=1,flags=re.S),end='')") \
     <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('skills/planning/SKILL.md').read(),count=1,flags=re.S),end='')")
grep -nE '\b(opus|sonnet)\b|AskUserQuestion|EnterWorktree|scratchpad|code-review' skills/planning/SKILL.md || echo "clean"
grep -nw Explore skills/planning/SKILL.md || echo "clean"
```
Expected: correct frontmatter; diff shows only the fallback insertion; `clean` twice.

- [ ] **Step 5: Delete old command and commit**

```bash
git rm commands/planning.md
git add skills/planning/SKILL.md
git commit -m "feat: convert planning command to a skill"
```

---

### Task 4: Convert the building command to a skill

**Files:**
- Create: `skills/building/SKILL.md`
- Delete: `commands/building.md`

- [ ] **Step 1: Verify absence (expect failure)**

Run:
```bash
test -f skills/building/SKILL.md && echo EXISTS || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Create the skill file**

```bash
mkdir -p skills/building
cat > skills/building/SKILL.md <<'EOF'
---
name: building
description: Implement signed-off issues end to end with a team of agents in an isolated worktree. Before coding, a generator and an adversarial evaluator negotiate a granular contract of what "done" means; after coding, the evaluator black-box tests the running artifact against that contract until it passes. Ships as a PR.
disable-model-invocation: true
---
EOF
python3 -c "import re,sys; sys.stdout.write(re.sub(r'^---\n.*?\n---\n','',open('commands/building.md').read(),count=1,flags=re.S))" >> skills/building/SKILL.md
```

- [ ] **Step 3: Insert the argument-fallback line**

Insert immediately after the `Requested work:\n$ARGUMENTS` block:
```bash
python3 - <<'PY'
p='skills/building/SKILL.md'; s=open(p).read()
anchor='Requested work:\n$ARGUMENTS\n'
add=anchor+'\nIf `$ARGUMENTS` is empty or was not substituted, treat the user\'s request itself as the arguments.\n'
assert anchor in s, 'anchor not found'
open(p,'w').write(s.replace(anchor,add,1))
PY
```

- [ ] **Step 4: Verify**

```bash
head -5 skills/building/SKILL.md
diff <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('commands/building.md').read(),count=1,flags=re.S),end='')") \
     <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('skills/building/SKILL.md').read(),count=1,flags=re.S),end='')")
grep -nE '\b(opus|sonnet)\b|AskUserQuestion|EnterWorktree|scratchpad|code-review' skills/building/SKILL.md || echo "clean"
grep -nw Explore skills/building/SKILL.md || echo "clean"
```
Expected: correct frontmatter; diff shows only the fallback insertion; `clean` twice.

- [ ] **Step 5: Delete old command and commit**

```bash
git rm commands/building.md
git add skills/building/SKILL.md
git commit -m "feat: convert building command to a skill"
```

---

### Task 5: Add the Pi package manifest

**Files:**
- Create: `package.json`

- [ ] **Step 1: Verify absence (expect failure)**

Run:
```bash
test -f package.json && echo EXISTS || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Read the current plugin metadata to copy values**

```bash
cat .claude-plugin/plugin.json   # copy name, version, description verbatim
```

- [ ] **Step 3: Write package.json**

Use the `name`, `version`, `description` from `.claude-plugin/plugin.json`:
```bash
cat > package.json <<'EOF'
{
  "name": "forge",
  "version": "1.3.0",
  "description": "Idea-to-PR pipeline with a human gate at each seam: grill an idea into a spec, plan it adversarially into GitHub issues, then implement it against a negotiated contract and ship a PR.",
  "keywords": ["pi-package", "workflow", "planning", "github", "issues", "pull-request", "agents"],
  "pi": { "skills": ["./skills"] }
}
EOF
```
(If `plugin.json` version differs from `1.3.0`, use the actual value.)

- [ ] **Step 4: Verify it parses and points at real skills**

```bash
python3 -c "import json;d=json.load(open('package.json'));print('pi.skills:',d['pi']['skills']); import os;print('dir exists:',os.path.isdir('skills'))"
```
Expected: `pi.skills: ['./skills']` and `dir exists: True`.

- [ ] **Step 5: Commit**

```bash
git add package.json
git commit -m "feat: add Pi package manifest declaring the skills directory"
```

---

### Task 6: Update Claude Code manifests and README wording

Change "commands" to "skills" in human-facing text; do not change install mechanics.

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`
- Modify: `README.md`

- [ ] **Step 1: Find every "command" mention to review**

```bash
grep -rniE 'command' .claude-plugin/ README.md
```
Expected: a list of lines to review (descriptions, the requirements/usage sections, the commands table).

- [ ] **Step 2: Edit each to say "skills"**

For each hit, reword "command"/"commands" to "skill"/"skills" where it refers to forge's three workflows. Update the README usage/table so it shows the three skills and their invocation (`/forge:interview`, `/forge:planning`, `/forge:building`). Do NOT change the `/plugin marketplace add` / `/plugin install` lines. Do NOT introduce an em dash.

- [ ] **Step 3: Verify no stale "command" references to the workflows remain, and JSON still parses**

```bash
python3 -c "import json;json.load(open('.claude-plugin/plugin.json'));json.load(open('.claude-plugin/marketplace.json'));print('json ok')"
grep -rniE 'command' .claude-plugin/ README.md || echo "no command references remain"
grep -c '—' README.md .claude-plugin/*.json   # expect 0 each
```
Expected: `json ok`; either no command references or only ones you deliberately kept (e.g. a `gh` shell command reference), and zero em dashes.

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json README.md
git commit -m "docs: describe forge as skills, not commands"
```

---

### Task 7: Full structural verification pass

Prove the conversion is complete and consistent before any live test.

**Files:** none (verification only)

- [ ] **Step 1: Assert no commands directory and three valid skills exist**

```bash
test -d commands && echo "FAIL: commands/ still present" || echo "OK: no commands/"
for w in interview planning building; do
  f=skills/$w/SKILL.md
  test -f "$f" && echo "OK: $f" || echo "FAIL: $f missing"
  head -4 "$f" | grep -q 'name:' && head -4 "$f" | grep -q 'description:' && head -4 "$f" | grep -q 'disable-model-invocation:' && echo "  frontmatter OK" || echo "  FAIL frontmatter"
done
```
Expected: `OK: no commands/`, and each skill file present with `frontmatter OK`.

- [ ] **Step 2: Assert no forbidden tokens anywhere in skills/**

```bash
grep -rnIE '\b(opus|sonnet)\b|AskUserQuestion|EnterWorktree|scratchpad|code-review' skills/ | grep -v disable-model-invocation || echo "clean"
grep -rnIw Explore skills/ || echo "clean"
```
Expected: `clean` twice.

- [ ] **Step 3: Assert bodies match the pre-conversion source**

```bash
git fetch -q origin building/issue-3 2>/dev/null || true
for w in interview planning building; do
  echo "=== $w ==="
  diff <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('/dev/stdin').read(),count=1,flags=re.S),end='')" < <(git show building/issue-3:commands/$w.md)) \
       <(python3 -c "import re;print(re.sub(r'^---\n.*?\n---\n','',open('skills/$w/SKILL.md').read(),count=1,flags=re.S),end='')") \
    && echo "identical" || echo "only-fallback-diff (inspect: must be exactly the one added line)"
done
```
Expected: each shows only the single added fallback line versus the `building/issue-3` source.

- [ ] **Step 4: Commit a marker if any fixups were needed**

If steps 1-3 revealed a defect, fix it in the offending task's file and commit `fix: ...`. If all passed, no commit needed.

---

### Task 8: Claude Code behavioral parity check

Confirm the three skills invoke and behave like the old commands on Claude Code.

**Files:** none (live check)

- [ ] **Step 1: Confirm discovery**

In a Claude Code session with this plugin/worktree active, list available skills and confirm `forge:interview`, `forge:planning`, `forge:building` appear and are invocable.

- [ ] **Step 2: Invoke the lightest skill with an argument**

Run `/forge:interview a CLI that renames photos by EXIF date` and confirm: it is invoked explicitly (not auto-fired), `$ARGUMENTS` arrived (it interviews about that idea), and it follows the phases. Do NOT complete a full run; confirming correct start is enough.

- [ ] **Step 3: Confirm explicit-only**

In normal conversation, mention "I might build something" WITHOUT invoking a skill; confirm no forge skill auto-fires (validates `disable-model-invocation`).

- [ ] **Step 4: Record the result**

Note pass/fail per skill in the PR description when opened. No commit.

---

### Task 9: Pi behavioral validation

Resolve the two Pi open items by running on a real Pi environment.

**Files:** none (live check on Pi)

- [ ] **Step 1: Install the package on Pi**

In the Pi environment, install/point at this package so Pi discovers `pi.skills`. Confirm the three skills are discoverable via `/skill:` invocation.

- [ ] **Step 2: Invoke a skill and observe argument handling (open item 1)**

Invoke the building skill on a small issue with a flag, e.g. request "run the forge building skill on issue 3 with --no-gate". Observe whether Pi passed `$ARGUMENTS` (substituted) or the model extracted the issue/flag from the request via the fallback line. Record which path worked.

- [ ] **Step 3: Confirm frontmatter tolerance (open item 2)**

Confirm the skill loaded without error despite the `disable-model-invocation` field (i.e. Pi ignores unknown fields rather than failing). Record the behavior.

- [ ] **Step 4: Confirm binding resolution**

Confirm the skill read `docs/harness-bindings/pi.md` and resolved the neutral verbs (used GLM-5.2 for the roles, git worktree, etc.). This repeats the manual command-form validation for the skill form.

- [ ] **Step 5: Record results**

Write the observed answers to open items 1 and 2, plus pass/fail, into the PR description. If argument handling or field tolerance failed, open a follow-up note; do not block the Claude Code side.

---

## Self-Review

**Spec coverage:** structure (Tasks 2-5), frontmatter (Tasks 2-4), arguments + fallback (Tasks 2-4), manifests (Tasks 5-6), what-does-not-change enforced by the body-parity checks (Tasks 2-4, 7), invocation model (Tasks 8-9), all four open items (Task 1 covers naming + agentskills; Task 9 covers Pi args + `disable-model-invocation`), out-of-scope respected (no `.pi/extensions/*` task), testing (Tasks 7-9), risks (naming risk gated by Task 1; Pi-arg risk gated by Task 9; marketplace risk checked in Task 6/8). No gaps found.

**Placeholder scan:** the only deferred value is `SKILL_NAME_PATTERN` from Task 1, which is a real task-produced interface with a concrete default, not a placeholder. All commands and file contents are literal.

**Type consistency:** skill directory names `skills/<workflow>/` (bare, per Task 1), `name:` values `<workflow>` (or Task 1's resolved value), and invocation `/forge:<workflow>` are used consistently across Tasks 1-9.
