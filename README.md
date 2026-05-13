# goobreview

Verification-first multi-agent code review for [Claude Code](https://www.anthropic.com/claude-code). A local mirror of Anthropic's `/ultrareview` with stronger guarantees:

- **One Opus 4.7 max-reasoning agent per execution path** (entry point + its reachable call chain)
- **Mandatory reproduction**: every reported concern is reproduced in an isolated git worktree before reporting
- **No fixes applied**: report-only, with proposed-fix diffs ready for another agent or a human

If a concern can't be reproduced, it doesn't make the report.

## Install

Paste this into Claude Code:

> Clone `https://github.com/Peaches99/goobreview` into `~/.claude/skills/goobreview` so the `/goobreview` skill becomes available.

Or run in your shell:

```bash
git clone https://github.com/Peaches99/goobreview.git ~/.claude/skills/goobreview
```

The skill is auto-discovered by Claude Code on next session.

## Usage

```
/goobreview              # diff vs default branch
/goobreview 1234         # GitHub PR #1234 (needs gh CLI)
/goobreview src/auth     # every entry point reachable from a path
/goobreview --all        # whole codebase (confirms cost first)
```

Output writes to `goobreview-report-<timestamp>.md` (+ `.json`) in the current directory. No source files are modified.

## How it works

```
Scope → Path discovery → Phase 1: parallel analyzers (Opus 4.7, one per path) →
Dedup candidates → Phase 2: parallel verifiers (Opus 4.7, worktree-isolated, one per concern) →
Aggregate VERIFIED only → Markdown + JSON report
```

Every spawned agent runs `model: opus` with the `ultrathink` keyword for max reasoning depth. Verifiers use `isolation: "worktree"` so each gets a fresh git worktree to install deps and run tests in. The worktree auto-cleans when the verifier finishes.

## Caveats

- **Cost is real.** A medium PR spawns 20+ Opus 4.7 max-reasoning agents. The skill prints an estimate before launching but does not enforce a budget.
- **Verification asymmetry.** Races, perf regressions, and memory leaks tend to come back INCONCLUSIVE rather than VERIFIED. goobreview is strongest on logic, null, type, injection, and auth bugs.
- **Heuristic path discovery.** Dynamic dispatch (reflection, plugin loaders, eval) can hide entry points. Use `<path>` mode to override.
- **Untested at scale.** Try on a small PR first to surface friction.

## Files

- `SKILL.md` — orchestration logic + analyzer/verifier prompts
- `lib/path-discovery.md` — entry-point heuristics per language
- `lib/repro-strategies.md` — verification recipes for 25 bug classes
- `lib/severity-rubric.md` — severity definitions

## License

MIT
