# debloat plugin (Copilot CLI) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Copilot-CLI-only plugin containing a scanner agent (read+search+execute, no edit) and an applicator agent (adds edit) that together replace the single-agent `debloat` skill, adding complexity flagging and a checkbox-report handoff between scanning and applying.

**Architecture:** A real plugin directory (`.claude-plugin/plugin.json` + `skills/` + `agents/`), developed as its own git repo. Two skills hold the detailed procedure (scanner: detect and report; applicator: read approved items and apply them under the existing safety workflow). Two agents are thin wrappers around those skills that carry the actual tool-grant difference between them.

**Tech Stack:** Markdown (SKILL.md and `.agent.md` files with YAML frontmatter). No custom code. Verified with a synthetic demo repo, same style of dogfooding used for the original `debloat` skill and `session-memory`.

## Global Constraints

- Copilot CLI only — no Claude Code packaging in this repo.
- Scanner agent tools: `[read, search, execute]` — no `edit`, ever.
- Applicator agent tools: `[read, search, execute, edit]` — the only agent that ever modifies files.
- Never auto-applies — applicator always ends in a presented diff awaiting explicit approval, even for checked/approved report items.
- Applicator only acts on `- [x]` checked items; unchecked items and any `COMPLEXITY-*` item are always skipped regardless of checkbox state.
- Complexity flagging never proposes a rewrite, under any circumstance — report-only.
- Repo developed locally at `~/.agents/plugins/debloat/`; GitHub remote added once the user provides the repo URL (external dependency, not blocking this plan's tasks).
- Old `debloat` skill retired from Copilot's discovery only, by relocating it out of the shared `~/.agents/skills/` into a Claude-Code-only real directory — Claude Code keeps working, Copilot stops seeing it.

---

### Task 1: Scaffold the plugin repo

**Files:**
- Create: `/Users/invoice4u/.agents/plugins/debloat/.gitignore`
- Create: `/Users/invoice4u/.agents/plugins/debloat/README.md`
- Create: `/Users/invoice4u/.agents/plugins/debloat/.claude-plugin/plugin.json`

**Interfaces:**
- Produces: the manifest Copilot CLI's `plugin install` reads, and the git repo every later task commits into.

- [ ] **Step 1: Initialize git and wire up the remote**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git init -q -b main
git config user.name "Ishay"
git config user.email "ishay@invoice4u.co.il"
git remote add origin https://github.com/idshay16/Code-Debloater-Agent.git
git remote -v
```

Expected: `origin` listed twice (fetch/push) pointing at
`https://github.com/idshay16/Code-Debloater-Agent.git`. This assumes the
repo already exists on GitHub (created by the user) — if `git remote -v`
succeeds but a later push fails with "repository not found," stop and
confirm the repo exists before continuing.

- [ ] **Step 2: Write .gitignore**

```
.debloat/
node_modules/
*.log
```

- [ ] **Step 3: Write plugin.json**

```json
{
  "$schema": "https://anthropic.com/claude-code/plugin.schema.json",
  "name": "debloat",
  "version": "0.1.0",
  "description": "Scanner/applicator agents for repo bloat cleanup: dead code, dead attributes, redundant logging, comment spam, and complexity flagging.",
  "author": {
    "name": "Ishay",
    "email": "ishay@invoice4u.co.il"
  },
  "skills": [
    "./skills"
  ]
}
```

- [ ] **Step 4: Write README.md**

```markdown
# Code-Debloater-Agent

Copilot CLI plugin: a scanner agent that finds bloat and complexity issues
in a repo, and an applicator agent that applies only the findings you've
approved. Copilot CLI only — see `DESIGN.md` for why.

## Install on a new machine

```bash
copilot plugin install idshay16/Code-Debloater-Agent
```

That's it — no cloning needed. Copilot CLI pulls the plugin directly from
GitHub and it persists across sessions on that machine. Confirm it worked:

```bash
copilot plugin list        # should show "Code-Debloater-Agent" (or "debloat")
copilot skill list          # should show debloat-scanner and debloat-applicator
```

### Local development install

When iterating on this repo locally instead of installing the published
version:

```bash
copilot plugin install /absolute/path/to/your/local/clone
```

Note: GitHub Copilot CLI currently flags direct local-path/repo/URL
installs as deprecated in favor of marketplace installs — this works today
but may need revisiting in a future CLI release. `copilot plugin install
<owner>/<repo>` (the GitHub-repo form used above) is the non-deprecated
path and is what other machines should use.

## Usage

1. Invoke the scanner agent to sweep a repo (batch mode, directory by
   directory, or flow mode scoped to a test-covered flow). It writes a
   checkbox report to `.debloat/reports/<date>-<scope>.md` in the target
   repo — never edits anything.
2. Review the report. Check the items you want applied (`- [x]`). Leave
   `COMPLEXITY-*` items as informational — they're never applied, only
   flagged.
3. Invoke the applicator agent. It reads only the checked items, works in
   an isolated branch, runs whatever tests cover the touched area, and
   presents a diff for your approval before anything is considered final.

See `DESIGN.md` for the full design rationale, including why detection has
shell access but enforcement is instruction-level rather than a hard
sandbox.
```

- [ ] **Step 5: Verify manifest is valid JSON**

```bash
python3 -c "import json; json.load(open('/Users/invoice4u/.agents/plugins/debloat/.claude-plugin/plugin.json')); print('MANIFEST_OK')"
```

Expected: `MANIFEST_OK`

- [ ] **Step 6: Commit**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git add .gitignore README.md .claude-plugin/plugin.json DESIGN.md PLAN.md
git commit -q -m "Scaffold debloat plugin: manifest, README, design docs"
echo COMMIT_OK
```

Expected: `COMMIT_OK`

---

### Task 2: Write the scanner skill

**Files:**
- Create: `/Users/invoice4u/.agents/plugins/debloat/skills/scanner/SKILL.md`

**Interfaces:**
- Produces: the detection/reporting procedure `agents/scanner.agent.md` (Task 4) delegates to.

- [ ] **Step 1: Write the complete scanner SKILL.md**

```markdown
---
name: debloat-scanner
description: >
  Sweeps a repository for bloat and complexity issues — dead code, dead
  object attributes, redundant/useless logging, comment spam from
  agent-assisted fixes, and complexity flags (deep nesting, long functions,
  duplicated logic, tangled control flow) — and writes a checkbox report.
  Never edits anything; that's the applicator's job. Trigger phrases: "scan
  this repo for bloat", "find dead code", "check complexity", "debloat scan".
---

# Debloat Scanner

Detects bloat and complexity issues, writes a checkbox report. Never edits,
writes, or deletes anything — that boundary is absolute regardless of what
tool access is technically available.

## Categories

1. **Dead code** — unreachable functions, unused exports/imports/variables.
2. **Dead object attributes** — fields set but never meaningfully read (or
   vice versa). Always flagged with extra caution regardless of tool
   support — cross-file field usage is the category most prone to false
   positives, especially in dynamic-dispatch-heavy code (PHP/CMS hooks,
   magic methods, reflection).
3. **Redundant/useless logging** — leftover debug statements, duplicate
   logging of the same information, logging with no diagnostic value.
4. **Comment spam from agent fixes** — narration comments describing what
   changed rather than why something non-obvious exists (e.g. "fixed bug
   where X", "updated because Y"), stale commented-out code blocks.
5. **Complexity flags** (report-only, never a removal candidate): deep
   nesting, long functions, high cyclomatic complexity, god
   objects/classes, duplicated/near-duplicated logic across files, tangled
   control/data flow (heavy coupling, shared mutable state, unpredictable
   branching).

## Detection method

Discover the repo's language/tooling first (check for `package.json`,
`requirements.txt`/`pyproject.toml`, `composer.json`, etc.).

- Categories 1-2 and the structural-metrics half of category 5: prefer real
  static analysis when present — `npx eslint` (with the `complexity` rule
  for category 5) or `npx ts-prune` for JS/TS, `python3 -m vulture` /
  `python3 -m pyflakes` / `radon` for Python, `phpstan`/`psalm`/`phpmd` for
  PHP, or the equivalent for the repo's language. Verify the tool is
  actually invocable (`which <tool>` or `npx --no-install <tool>
  --version`) before relying on it. Fall back to agent judgment — read the
  code, trace references by grep — when nothing is available.
- Categories 3-4 and the duplicated-logic/tangled-flow half of category 5:
  always agent judgment, no static tool covers these.

Every finding gets a one-line rationale and a confidence tag:
`tool-confirmed` (a real static analysis tool flagged it) or
`agent-judgment` (reasoning-based, no tool backing).

## Operating modes

**Batch mode**: sweep the repo directory-by-directory (or module-by-module).
If a directory's findings would be too many to review comfortably in one
report section, split it further — judge this the way a human would decide
a PR is too big, no fixed threshold.

**Flow mode**: identify a flow either by running the test suite with
coverage instrumentation and using the resulting map, or by the user naming
a flow directly ("scan the checkout flow") and tracing it by
reading/grepping to find the relevant code and its test(s). Code outside
any covered flow has no safety net for later application — note this in
the report so the applicator (and the user) know which findings are
backed by tests and which aren't.

## Report format

Write to `.debloat/reports/<YYYY-MM-DD>-<scope>.md` in the target repo
(create the `.debloat/reports/` directory if it doesn't exist):

```markdown
# Debloat scan report — <repo> — <date>

## Batch/flow: <scope>

- [ ] **DEAD-1** `path/to/file.ext:LINE` — <what and why unreferenced>
  (tool-confirmed|agent-judgment)
- [ ] **ATTR-1** `path/to/file.ext:LINE` — <field name> set but never read
  (agent-judgment — always this tag for this category)
- [ ] **LOG-1** `path/to/file.ext:LINE-LINE` — <what's redundant>
  (tool-confirmed|agent-judgment)
- [ ] **COMMENT-1** `path/to/file.ext:LINE` — <narration comment or stale
  block> (agent-judgment)
- [ ] **COMPLEXITY-1** `path/to/file.ext:LINE-LINE` — <metric/signal and
  value> (tool-confirmed|agent-judgment) — flagged for attention, not a
  removal candidate

## Coverage notes

<which parts of this batch/flow have test coverage, which don't>
```

IDs are sequential per category within one report (`DEAD-1`, `DEAD-2`,
...), not globally unique across reports — the applicator identifies items
by report file + ID together.

## Boundaries

- Never edit, write, or delete a file, or run a mutating git command
  (commit, push, checkout -b then modify, etc.) — regardless of what shell
  access is technically available. If asked to "just fix it" mid-scan,
  decline and point to the applicator instead.
- Never propose a rewrite for a complexity flag, including on a specific
  follow-up request for that one item.
- Never mark a dead-object-attribute finding as anything but
  extra-cautious, regardless of tool confirmation.
- One repo per invocation — never batch across multiple repos in a single
  sweep.
```

- [ ] **Step 2: Verify frontmatter structure**

```bash
f=/Users/invoice4u/.agents/plugins/debloat/skills/scanner/SKILL.md
head -1 "$f" | grep -qx -- '---' && echo "OPENS: PASS" || echo "OPENS: FAIL"
sed -n '2,20p' "$f" | grep -q '^name: debloat-scanner$' && echo "NAME: PASS" || echo "NAME: FAIL"
grep -inE 'TBD|TODO|FIXME|xxx' "$f" || echo NO_PLACEHOLDERS
```

Expected: `OPENS: PASS`, `NAME: PASS`, `NO_PLACEHOLDERS`

- [ ] **Step 3: Commit**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git add skills/scanner/SKILL.md
git commit -q -m "Add scanner skill: detection logic and report format"
echo COMMIT_OK
```

---

### Task 3: Write the applicator skill

**Files:**
- Create: `/Users/invoice4u/.agents/plugins/debloat/skills/applicator/SKILL.md`

**Interfaces:**
- Consumes: report file format produced by Task 2's scanner skill (checkbox items with IDs, category prefixes, confidence tags).
- Produces: the apply procedure `agents/applicator.agent.md` (Task 4) delegates to.

- [ ] **Step 1: Write the complete applicator SKILL.md**

```markdown
---
name: debloat-applicator
description: >
  Reads an approved debloat scan report and applies only the checked
  findings, under a strict safety workflow — isolated branch, test
  verification, diff review, never auto-applies. Trigger phrases: "apply
  the debloat report", "apply approved findings", "run the applicator".
---

# Debloat Applicator

Applies only what's been explicitly approved in a scanner report. Never
auto-applies a change — every batch ends in a presented diff awaiting
explicit approval, even for items already checked in the report.

## Reading the report

1. Locate the report file (most recent under `.debloat/reports/` unless
   the user names a specific one).
2. Parse each line: only `- [x]` (checked) items are candidates.
   `- [ ]` (unchecked) items are skipped — the user chose not to approve
   them.
3. **`COMPLEXITY-*` items are never actionable, regardless of checkbox
   state.** They're informational only; if one is checked, that's not a
   signal to act on it — flag this to the user rather than silently
   ignoring or silently acting.
4. If the report's coverage notes indicate a finding is in an area with no
   test coverage, carry that forward into the diff presentation (see
   Safety workflow, step 4) — don't drop the distinction.

## Safety workflow

1. Work in an isolated git branch/worktree — use
   `superpowers:using-git-worktrees` rather than improvising isolation.
2. Apply only the checked, non-`COMPLEXITY-*` findings from the report.
3. Run whatever tests cover the touched area.
4. Present the diff. Every change keeps its rationale and confidence tag
   from the report. If the report's coverage notes marked the area as
   uncovered, say so explicitly here too — mark it unverified, don't
   present it with the same confidence framing as a tested change.
5. Wait for explicit approval before treating the change as final — this
   holds even though the report already marked these items as approved;
   the report is a scoping input, not a substitute for reviewing the
   actual diff.
6. Once approved, hand off to `superpowers:finishing-a-development-branch`
   for integration (merge/PR/keep-as-is).

## Boundaries

- Never auto-apply. Checkbox approval in the report narrows *what* to
  attempt, it does not skip the diff-review gate.
- Never act on an unchecked item.
- Never act on a `COMPLEXITY-*` item under any circumstances — flag it back
  to the user if one is checked, don't treat the checkbox as authorization
  to rewrite anything.
- If the report file can't be found or parsed, stop and ask — don't guess
  at what was approved.
- One repo per invocation.
```

- [ ] **Step 2: Verify frontmatter structure**

```bash
f=/Users/invoice4u/.agents/plugins/debloat/skills/applicator/SKILL.md
head -1 "$f" | grep -qx -- '---' && echo "OPENS: PASS" || echo "OPENS: FAIL"
sed -n '2,20p' "$f" | grep -q '^name: debloat-applicator$' && echo "NAME: PASS" || echo "NAME: FAIL"
grep -inE 'TBD|TODO|FIXME|xxx' "$f" || echo NO_PLACEHOLDERS
```

Expected: `OPENS: PASS`, `NAME: PASS`, `NO_PLACEHOLDERS`

- [ ] **Step 3: Commit**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git add skills/applicator/SKILL.md
git commit -q -m "Add applicator skill: report parsing and safety workflow"
echo COMMIT_OK
```

---

### Task 4: Write the two agent files

**Files:**
- Create: `/Users/invoice4u/.agents/plugins/debloat/agents/scanner.agent.md`
- Create: `/Users/invoice4u/.agents/plugins/debloat/agents/applicator.agent.md`

**Interfaces:**
- Consumes: `skills/scanner/SKILL.md` (Task 2) and `skills/applicator/SKILL.md` (Task 3) by name reference.
- Produces: the two invocable agents Task 5 installs and tests.

- [ ] **Step 1: Write scanner.agent.md**

```markdown
---
name: debloat-scanner
description: Scans a repository for bloat and complexity issues, writes a checkbox report. Read-only — never edits, writes, or deletes.
tools: [read, search, execute]
---

You are the debloat scanner. Your only job is detection and reporting —
you never modify anything, no matter how you're asked.

Follow the `debloat-scanner` skill's procedure in full: discover the
repo's language and tooling, run in batch mode (directory by directory) or
flow mode (coverage-driven or a named flow) as the user specifies, detect
all 5 categories (dead code, dead attributes, redundant logging, comment
spam, complexity flags), and write the checkbox report to
`.debloat/reports/<date>-<scope>.md`.

You have shell access (`execute`) for running real static analysis tools
and, in flow mode, instrumenting test coverage. That access is for
detection only. You do not have an edit tool, and you must never use shell
access to write, delete, or modify any file, or to run a mutating git
command (commit, push, add+checkout that changes tracked state, etc.). If
asked to fix or apply something mid-scan, decline and say that's the
applicator agent's job — you only report.

When you finish a scan, tell the user where the report is and how many
findings are in each category, then stop. Applying anything is out of
scope for you.
```

- [ ] **Step 2: Write applicator.agent.md**

```markdown
---
name: debloat-applicator
description: Reads an approved debloat scan report and applies only the checked findings, under a strict safety workflow that never auto-applies.
tools: [read, search, execute, edit]
---

You are the debloat applicator. You are the only one of the two debloat
agents with edit access, and you use it under a strict process, not
freely.

Follow the `debloat-applicator` skill's procedure in full: locate the scan
report, act only on checked (`- [x]`) findings, never act on unchecked
items or on any `COMPLEXITY-*` item regardless of its checkbox state, work
in an isolated branch via `using-git-worktrees`, run whatever tests cover
the touched area, and present a diff with rationale and confidence for
every change — including calling out anything the report marked as
untested. Wait for explicit approval before treating anything as final,
then hand off to `finishing-a-development-branch`.

The report tells you what's in scope. It does not substitute for the
diff-review gate — you present the actual diff and wait, every time, even
though the items were already checked as approved in the report.
```

- [ ] **Step 3: Verify both files' frontmatter**

```bash
for f in /Users/invoice4u/.agents/plugins/debloat/agents/scanner.agent.md /Users/invoice4u/.agents/plugins/debloat/agents/applicator.agent.md; do
  echo "=== $f ==="
  head -1 "$f" | grep -qx -- '---' && echo "OPENS: PASS" || echo "OPENS: FAIL"
  grep -q '^name:' "$f" && echo "NAME: PASS" || echo "NAME: FAIL"
  grep -q '^tools:' "$f" && echo "TOOLS: PASS" || echo "TOOLS: FAIL"
done
grep -l 'tools: \[read, search, execute\]$' /Users/invoice4u/.agents/plugins/debloat/agents/scanner.agent.md && echo "SCANNER_NO_EDIT: PASS"
grep -l 'tools: \[read, search, execute, edit\]$' /Users/invoice4u/.agents/plugins/debloat/agents/applicator.agent.md && echo "APPLICATOR_HAS_EDIT: PASS"
```

Expected: all `PASS` lines, confirming the scanner's tools line has no
`edit` and the applicator's does.

- [ ] **Step 4: Commit**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git add agents/scanner.agent.md agents/applicator.agent.md
git commit -q -m "Add scanner and applicator agent definitions"
echo COMMIT_OK
```

---

### Task 5: Local install and dogfood verification

**Files:**
- Create (temporary, deleted at end of task): a synthetic demo git repo under the scratchpad directory
- Modify (cleanup): uninstalls the local-path plugin install and removes the demo repo

**Interfaces:**
- Consumes: the full plugin directory from Tasks 1-4.
- Produces: confirmation that `copilot plugin install` recognizes the plugin and that the scanner's report format / applicator's checkbox-gating logic behave as designed, using the same style of hand-applied verification used for the original `debloat` skill and `session-memory` (no CI, no test framework — this is a documentation-and-agent-definition deliverable).

- [ ] **Step 1: Install the plugin locally and confirm discovery**

```bash
copilot plugin install /Users/invoice4u/.agents/plugins/debloat 2>&1
copilot plugin list 2>&1 | grep -i debloat
```

Expected: install succeeds (the same "deprecated" warning seen during
design's probe test is expected and fine for local dev), and `debloat`
appears in the plugin list.

- [ ] **Step 2: Confirm both skills are visible**

```bash
cd /Users/invoice4u/Desktop/Invoice4u/MobileApp/invoice4u_mobile_app
copilot skill list 2>&1 | grep -i debloat
```

Expected: both `debloat-scanner` and `debloat-applicator` appear.

- [ ] **Step 3: Build a synthetic demo repo covering all 5 categories**

```bash
DEMO=/private/tmp/claude-501/-Users-invoice4u/f21e6fdf-9751-4ad4-b5ad-b6aa1af83e30/scratchpad/debloat-plugin-demo
rm -rf "$DEMO"
mkdir -p "$DEMO"
cd "$DEMO"
git init -q
cat > app.js <<'EOF'
class Widget {
  constructor(name) {
    this.name = name;
    this.internalCounter = 0; // dead attribute
  }

  render() {
    // fixed bug where render crashed on empty name — see ticket #412
    if (this.name) {
      if (this.name.length > 0) {
        if (!this.name.includes(" ")) {
          console.log("rendering widget:", this.name);
          console.log("rendering widget:", this.name);
        } else {
          console.log("rendering widget with space:", this.name);
        }
      }
    }
    return `<div>${this.name}</div>`;
  }

  // legacyRenderer(name) {
  //   return "<span>" + name + "</span>";
  // }
}

function unusedHelper() {
  return 42;
}

module.exports = { Widget };
EOF
git add -A
git commit -q -m "initial demo commit"
echo DEMO_REPO_CREATED
```

Expected: `DEMO_REPO_CREATED`

- [ ] **Step 4: Hand-apply the scanner's detection heuristics against the demo repo, covering all 5 categories**

```bash
cd "$DEMO"
echo "--- DEAD: unusedHelper ---"
grep -c "unusedHelper" app.js
echo "--- ATTR: internalCounter ---"
grep -n "internalCounter" app.js
echo "--- LOG: duplicate console.log ---"
grep -n "console.log" app.js
echo "--- COMMENT: narration + stale block ---"
grep -n "fixed bug where\|legacyRenderer" app.js
echo "--- COMPLEXITY: nesting depth in render() ---"
sed -n '/render()/,/^  }/p' app.js | grep -c "if ("
```

Expected: `unusedHelper` found once (unreferenced), `internalCounter`
assigned once and never read, two duplicate `console.log` calls with
identical arguments, both the narration comment and the commented-out
block present, and 3 nested `if (` statements inside `render()` — matching
the scanner skill's dead code / dead attribute / redundant logging /
comment spam / complexity heuristics respectively.

- [ ] **Step 5: Hand-write the report exactly as the scanner skill's format specifies, to confirm the format is unambiguous**

```bash
mkdir -p "$DEMO/.debloat/reports"
cat > "$DEMO/.debloat/reports/2026-08-23-full-repo.md" <<'EOF'
# Debloat scan report — debloat-plugin-demo — 2026-08-23

## Batch/flow: full repo

- [x] **DEAD-1** `app.js:24` — `unusedHelper` unreferenced anywhere (agent-judgment)
- [ ] **ATTR-1** `app.js:4` — `internalCounter` set but never read (agent-judgment)
- [x] **LOG-1** `app.js:12-13` — duplicate console.log call (agent-judgment)
- [x] **COMMENT-1** `app.js:8` — narration comment describing a fix (agent-judgment)
- [x] **COMPLEXITY-1** `app.js:7-18` — render() has 3 levels of nested conditionals (agent-judgment) — flagged for attention, not a removal candidate

## Coverage notes

No test files in this repo — all findings above are unverified by tests.
EOF
echo REPORT_WRITTEN
```

Expected: `REPORT_WRITTEN`

- [ ] **Step 6: Hand-apply the applicator's gating logic and confirm it selects exactly the right subset**

```bash
cd "$DEMO"
echo "--- checked, non-COMPLEXITY items (should be applied) ---"
grep -E '^\- \[x\]' .debloat/reports/2026-08-23-full-repo.md | grep -v COMPLEXITY
echo "--- unchecked items (must be skipped) ---"
grep -E '^\- \[ \]' .debloat/reports/2026-08-23-full-repo.md
echo "--- checked COMPLEXITY item (must ALSO be skipped, despite being checked) ---"
grep -E '^\- \[x\]' .debloat/reports/2026-08-23-full-repo.md | grep COMPLEXITY
```

Expected: exactly `DEAD-1`, `LOG-1`, and `COMMENT-1` in the first group
(these are what the applicator would act on); `ATTR-1` in the second group
(skipped — unchecked); `COMPLEXITY-1` in the third group (skipped despite
being checked, per the applicator skill's explicit boundary that
`COMPLEXITY-*` is never actionable regardless of checkbox state).

- [ ] **Step 7: Clean up — uninstall the local plugin and remove the demo repo**

```bash
copilot plugin uninstall debloat 2>&1
rm -rf /private/tmp/claude-501/-Users-invoice4u/f21e6fdf-9751-4ad4-b5ad-b6aa1af83e30/scratchpad/debloat-plugin-demo
test ! -d /private/tmp/claude-501/-Users-invoice4u/f21e6fdf-9751-4ad4-b5ad-b6aa1af83e30/scratchpad/debloat-plugin-demo && echo CLEANUP_OK
```

Expected: `CLEANUP_OK`. (The plugin is reinstalled for real once the GitHub
repo exists — this local install was for verification only.)

- [ ] **Step 8: Push the verified content to GitHub**

```bash
cd /Users/invoice4u/.agents/plugins/debloat
git push -u origin main 2>&1
```

Expected: push succeeds. If it fails on authentication (this session has
no `gh` CLI and no working SSH identity loaded for GitHub, confirmed
during design — `ssh-add -l` returned "no identities" and `ssh -T
git@github.com` returned "Permission denied"), stop here and report the
exact error rather than retrying blindly — the user will need to push
manually from an authenticated shell, or fix auth in this environment
(e.g. `git config credential.helper osxkeychain` is already set, which may
prompt for a token/password on first HTTPS push rather than failing
outright — this is worth trying since it's different from the SSH path
already confirmed broken).

- [ ] **Step 9: Verify the real install path works (only if Step 8 succeeded)**

```bash
copilot plugin install idshay16/Code-Debloater-Agent 2>&1
copilot plugin list 2>&1 | grep -i debloat
copilot plugin uninstall Code-Debloater-Agent 2>&1
```

Expected: installs from the real GitHub repo (not a local path this time),
shows up in the list, then cleanly uninstalled — leaving it uninstalled is
intentional, so the user does a fresh, deliberate install afterward rather
than inheriting this session's test install silently. If the plugin name
registered by `plugin install` differs from `Code-Debloater-Agent` (it may
register under the manifest's internal `name: "debloat"` instead of the
repo name), check `copilot plugin list`'s actual output and uninstall
using whatever name it shows.

---

### Task 6: Retire the old `debloat` skill from Copilot's discovery, keep it for Claude Code

**Files:**
- Modify: relocates `/Users/invoice4u/.agents/skills/debloat/` to
  `/Users/invoice4u/.claude/skills/debloat/` as a real directory (not a
  symlink)
- Delete: the old symlink at `/Users/invoice4u/.claude/skills/debloat`

**Interfaces:**
- Consumes: nothing from earlier tasks — independent cleanup step.
- Produces: Copilot CLI (which reads `~/.agents/skills/` and
  `~/.copilot/skills/`, never `~/.claude/skills/`) stops seeing the old
  skill; Claude Code (which reads `~/.claude/skills/` directly) keeps
  seeing it as before, just as a real directory instead of a symlink
  target.

- [ ] **Step 1: Confirm current state before touching anything**

```bash
ls -la ~/.claude/skills/debloat
ls -la ~/.agents/skills/debloat
```

Expected: the first is a symlink pointing at
`../../.agents/skills/debloat`, the second is a real directory containing
`SKILL.md`, `DESIGN.md`, `PLAN.md`.

- [ ] **Step 2: Remove the symlink, move the real content in its place**

```bash
rm ~/.claude/skills/debloat
mv ~/.agents/skills/debloat ~/.claude/skills/debloat
```

- [ ] **Step 3: Verify Claude Code's copy is intact and real (not a symlink)**

```bash
test -f ~/.claude/skills/debloat/SKILL.md && echo "CLAUDE_SKILL_INTACT"
test ! -L ~/.claude/skills/debloat && echo "NO_LONGER_A_SYMLINK"
```

Expected: `CLAUDE_SKILL_INTACT`, `NO_LONGER_A_SYMLINK`

- [ ] **Step 4: Verify Copilot CLI no longer sees it**

```bash
cd /Users/invoice4u/Desktop/Invoice4u/MobileApp/invoice4u_mobile_app
copilot skill list 2>&1 | grep -i "^\s*debloat -" || echo "OLD_DEBLOAT_GONE_FROM_COPILOT"
```

Expected: `OLD_DEBLOAT_GONE_FROM_COPILOT` (the new `debloat-scanner`/
`debloat-applicator` skills from the plugin were already uninstalled in
Task 5 Step 7 for local-dev cleanup — they'll reappear once the real
GitHub-based install happens, which is outside this plan since it depends
on the user creating the repo).

- [ ] **Step 5: No commit — `~/.agents` and `~/.claude` are not git repos**

Skip, consistent with every prior skill built in this environment.
