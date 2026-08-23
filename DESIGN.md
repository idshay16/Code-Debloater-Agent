# debloat plugin — Design Spec (v3: Copilot CLI only, scanner/applicator split)

Date: 2026-08-23

## Scope: Copilot CLI only

This targets **GitHub Copilot CLI exclusively**. Claude Code is explicitly
out of scope — a Claude-Code-native version will be designed separately
later, since Claude Code's plugin subagents ignore the `hooks` field
(confirmed via docs.github.com/Claude Code docs research during design),
which rules out the enforcement mechanism that would make a shared design
worthwhile there. Building two different things per platform, each fitting
that platform's real capabilities, beats one compromised shared design.

The existing standalone skill at `~/.agents/skills/debloat/` (and its
`~/.claude/skills/debloat` symlink) stays in place for Claude Code — it is
**not** retired by this work, since Claude Code has no replacement yet. It
**is** retired from Copilot CLI's discovery once this plugin is built and
verified working (Copilot reads `~/.agents/skills/` directly, so the old
skill and the new plugin would otherwise both be offered there).

## Relationship to the prior `debloat` skill

All of the original single-skill design's substance carries forward: the 4
bloat categories, tool-assisted-detection-with-fallback, and the
never-auto-apply principle. See `~/.agents/skills/debloat/DESIGN.md` for
that original reasoning — restructured here, not replaced.

## What's new here

1. **Complexity flagging** — a 5th capability, structurally different from
   the 4 deletion categories: it never proposes a change, only reports.
2. **Split into two skills** — scanner (read-only detection, all 5
   capabilities) and applicator (takes an approved subset of the scanner's
   report and applies it).
3. **Wrapped in two custom agents** with different tool grants — not
   identical enforcement to a hard sandbox (see Enforcement below), but a
   real reduction in what the scanner can technically do, plus a
   consistently-reinforced behavioral boundary.
4. **Packaged as a real Copilot CLI plugin**, developed in its own git
   repository (see Repository below) rather than as loose files.

## Packaging & installation mechanics

Confirmed empirically during design (a throwaway probe plugin was
installed, listed, and cleanly uninstalled):

- `copilot plugin install <local-path>` accepts a plain absolute local
  directory path. GitHub's CLI flags direct local-path/repo/URL installs as
  **deprecated** in favor of marketplace installs — works today, may need
  revisiting if a future Copilot CLI release removes it. Since this will
  live in a real GitHub repo, `copilot plugin install owner/repo` (the
  non-deprecated GitHub-repo form) becomes the actual install path once
  pushed, rather than the local-path form used only for local dev/testing.
- Agent file schema (confirmed via docs.github.com/en/copilot/reference/
  custom-agents-configuration): `.agent.md` files under `agents/`, YAML
  frontmatter fields `name`, `description` (required), `tools`, `model`,
  `disable-model-invocation`, `user-invocable`, `mcp-servers`. Tool names
  accept Claude-style aliases (`Read`, `Grep`, `Glob`, `Bash`, `Edit`,
  `Write`) mapped onto Copilot's primary categories (`read`, `search`,
  `execute`, `edit`) — this design uses the primary category names
  directly. Body of the file is the agent's system prompt.

## Repository

A dedicated GitHub repo, created by the user (this session has no `gh` CLI
and no working GitHub SSH auth — confirmed, `ssh-add -l` returns "no
identities" and `ssh -T git@github.com` returns "Permission denied"),
suggested name `copilot-debloat-plugin`, private. The plugin is developed
there under normal git workflow (branch, commit, PR via
`finishing-a-development-branch`) rather than as an unversioned local
directory — this is real, shareable, iterable work, unlike the earlier
dotfile-based skills which had no repo to belong to.

Local development happens at `~/.agents/plugins/debloat/` (git-initialized,
remote added once the user creates the GitHub repo and shares its URL).

## Directory layout

```
~/.agents/plugins/debloat/            (= repo root)
  .git/
  .gitignore
  README.md
  DESIGN.md                           (this file)
  PLAN.md
  .claude-plugin/plugin.json          (manifest — Copilot CLI reads this)
  skills/
    scanner/SKILL.md
    applicator/SKILL.md
  agents/
    scanner.agent.md
    applicator.agent.md
```

## Bloat categories (unchanged from the original `debloat` design)

1. Dead code — unreachable functions, unused exports/imports/variables.
2. Dead object attributes — fields set but never meaningfully read (or vice
   versa); always flagged lower-confidence regardless of tool support.
3. Redundant/useless logging — leftover debug statements, duplicate
   logging, logging with no diagnostic value.
4. Comment spam from agent fixes — narration comments ("fixed bug where
   X"), stale commented-out code blocks.

## Complexity flagging (new)

Report-only, never proposes a rewrite — not even on a specific follow-up
request. Three signal types, all in scope:

- **Structural metrics** — tool-assisted where available (eslint
  `complexity` rule, `radon` for Python, `phpmd` for PHP, or equivalent):
  deep nesting, long functions, high cyclomatic complexity, god
  objects/classes.
- **Duplicated/near-duplicated logic** — agent judgment comparing code
  across the scan's scope.
- **Tangled control/data flow** — agent judgment: heavy coupling, shared
  mutable state, unpredictable branching.

Bundled into every scan — not a separately-triggered pass. Each flagged
item gets a one-line rationale and the same `tool-confirmed`/
`agent-judgment` confidence tag used for the deletion categories.

## Detection method (unchanged mechanics, now scanner-only)

Discover the repo's language/tooling first (same pattern as the `run`
skill). Prefer real static analysis where available, fall back to agent
judgment where not. Every finding — deletion candidate or complexity flag —
carries a one-line rationale and a confidence tag.

## Enforcement model (corrected during design — read before implementing)

Researched and confirmed: neither Copilot CLI's `.agent.md` `tools` field
nor (for plugin-packaged subagents specifically) Claude Code's subagent
frontmatter supports scoped/parameterized tool entries (no
`execute(git:*)`-style syntax, no deny-list alongside `tools`, no
per-command filtering). Restriction is category-granularity only:
`read`/`search`/`execute`/`edit` as whole units, nothing finer.

Given that, and the user's explicit choice to keep shell access on the
scanner for real tool-assisted detection:

- **`agents/scanner.agent.md`**: `tools: [read, search, execute]` — no
  `edit`. This is a real, technically-enforced restriction: the scanner
  cannot invoke Copilot's structured edit/write tool at all. It still has
  `execute` (shell), which could in principle run a write-capable command —
  that boundary is **instruction-only**: the system prompt explicitly and
  repeatedly states the scanner never writes, edits, deletes, or runs
  mutating git commands, and the design compensates by keeping the
  applicator as the sole path to any actual change (see Applicator below).
- **`agents/applicator.agent.md`**: `tools: [read, search, execute, edit]`
  — the only agent with `edit`, and the only one this design ever expects
  to modify files.

This is weaker than a hard sandbox and that's stated plainly, not papered
over — it is nonetheless a real improvement over a single undifferentiated
agent, both in reduced technical capability (no edit tool on the scanner)
and in the consistency of a dedicated, single-purpose system prompt per
role.

## Scanner skill / agent

Runs `skills/scanner/SKILL.md`'s logic: discover tooling, sweep in batch or
flow mode (same two modes as the original design — batch = directory by
directory, flow = coverage-driven or manually-named), and for each item
found (deletion candidate or complexity flag), write it as a checkbox entry
into a report file:

```markdown
# Debloat scan report — <repo> — <date>

## Batch/flow: <scope>

- [ ] **DEAD-1** `src/foo.js:42` — `unusedHelper` unreferenced anywhere
  (agent-judgment)
- [ ] **LOG-1** `src/widget.js:9-10` — duplicate `console.log` call
  (agent-judgment)
- [ ] **COMPLEXITY-1** `src/checkout.js:120-210` — `processOrder()` has
  cyclomatic complexity 24, 6 levels of nesting (tool-confirmed via eslint)
  — flagged for your attention, not proposed as a removal
```

Report location: written to the target repo (e.g.
`.debloat/reports/<date>-<scope>.md`, gitignored) so it's easy to find and
diff-review alongside the code it describes, rather than buried in a global
location disconnected from the repo it's about.

## Applicator skill / agent

Runs `skills/applicator/SKILL.md`'s logic:

1. Read the report file. Only act on checked (`- [x]`) items — anything
   left unchecked, or any `COMPLEXITY-*` item (report-only, never
   actionable by the applicator, regardless of checkbox state) is skipped.
2. Work in an isolated git branch/worktree (`using-git-worktrees`), same as
   the original design.
3. Apply only the approved deletions.
4. Run whatever tests cover the touched area.
5. Present the diff, same rationale/confidence framing as before, flag
   zero-coverage areas as unverified.
6. Wait for explicit approval before treating it as final, then hand off to
   `finishing-a-development-branch`.

Never auto-applies, exactly as before — the checkbox approval in the report
is necessary but not sufficient; the diff-review gate still stands.

## Out of scope for this revision

- No Claude Code packaging (see Scope above).
- No unified single-agent orchestration — the two-agent split is
  deliberate.
- No attempt to make the instruction-only shell boundary a provably-complete
  sandbox — documented as instruction-level, not a security guarantee.
- Everything already out-of-scope in the original `debloat` design still
  applies: no auto-apply mode, no cross-repo batching, no new
  workspace-isolation/integration mechanism beyond
  `using-git-worktrees`/`finishing-a-development-branch`.
