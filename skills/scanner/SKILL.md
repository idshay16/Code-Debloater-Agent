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
