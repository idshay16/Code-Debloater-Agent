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
