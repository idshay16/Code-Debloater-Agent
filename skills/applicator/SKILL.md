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
