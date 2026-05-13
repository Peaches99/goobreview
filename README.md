# goobreview

Verification-first multi-agent code review. A local mirror of Anthropic's
`/ultrareview`:

- **One high-reasoning agent per execution path** (entry point + reachable call chain)
- **Mandatory reproduction**: every reported concern is reproduced in an isolated git worktree before reporting
- **No fixes applied**: report-only, with proposed-fix diffs

If a concern can't be reproduced, it doesn't make the report.

Available for three coding agents:

| Platform | Directory | Parallelism | Worktree isolation |
|---|---|---|---|
| **Claude Code** | [`claude-code/`](claude-code/) | Native via `Agent` tool, fully parallel | Native via `isolation: "worktree"` |
| **Codex CLI** | [`codex/`](codex/) | Native subagents (default cap 6 concurrent) | Native (git worktrees) |
| **GitHub Copilot** | [`github-copilot/`](github-copilot/) | Sequential only | Manual via `git worktree add` |

The Claude Code and Codex versions are faithful ports of the architecture
(parallel analyzers + parallel verifiers in isolated worktrees). The Copilot
version runs the workflow sequentially because Copilot lacks native
sub-agent spawning — it's slower but the verification-first guarantee is
preserved.

---

## Install — Claude Code

**Paste into Claude Code:**

> Clone `https://github.com/Peaches99/goobreview` to a temp directory, then copy the `claude-code/` subfolder contents into `~/.claude/skills/goobreview/`, then delete the temp clone.

**Or one-liner:**

```bash
mkdir -p ~/.claude/skills/goobreview && \
  git clone https://github.com/Peaches99/goobreview /tmp/goobreview-install && \
  cp -r /tmp/goobreview-install/claude-code/. ~/.claude/skills/goobreview/ && \
  rm -rf /tmp/goobreview-install
```

Invoke with `/goobreview`, `/goobreview <PR#>`, `/goobreview <path>`, or
`/goobreview --all`. Auto-discovered on next Claude Code session.

---

## Install — Codex CLI

**Paste into Codex CLI:**

> Clone `https://github.com/Peaches99/goobreview` to a temp directory, then copy the `codex/` subfolder contents into `~/.codex/skills/goobreview/`, then delete the temp clone.

**Or one-liner:**

```bash
mkdir -p ~/.codex/skills/goobreview && \
  git clone https://github.com/Peaches99/goobreview /tmp/goobreview-install && \
  cp -r /tmp/goobreview-install/codex/. ~/.codex/skills/goobreview/ && \
  rm -rf /tmp/goobreview-install
```

Codex auto-discovers `SKILL.md` files in `~/.codex/skills/**`. Invoke by
asking Codex to "run goobreview" or similar natural-language prompts.

If you want true parallelism on large reviews, raise Codex's concurrent
thread cap in your config (default is 6).

---

## Install — GitHub Copilot

**Paste into Copilot Chat (agent mode):**

> Download `https://raw.githubusercontent.com/Peaches99/goobreview/main/github-copilot/goobreview.agent.md` and save it to `~/.copilot/agents/goobreview.agent.md` so the goobreview custom agent becomes available globally.

**Or one-liner (global, all repos):**

```bash
mkdir -p ~/.copilot/agents && \
  curl -fsSL https://raw.githubusercontent.com/Peaches99/goobreview/main/github-copilot/goobreview.agent.md \
    -o ~/.copilot/agents/goobreview.agent.md
```

**Or one-liner (workspace, one repo):**

```bash
mkdir -p .github/agents && \
  curl -fsSL https://raw.githubusercontent.com/Peaches99/goobreview/main/github-copilot/goobreview.agent.md \
    -o .github/agents/goobreview.agent.md
```

Select "goobreview" from the agents dropdown in Copilot Chat. Tell it the
scope (PR, path, or --all) and it runs the sequential workflow.

---

## Usage (all platforms)

```
goobreview              # diff vs default branch
goobreview 1234         # GitHub PR #1234 (needs gh CLI)
goobreview src/auth     # every entry point reachable from a path
goobreview --all        # whole codebase (confirms cost first)
```

Output writes to `goobreview-report-<timestamp>.md` (+ `.json`) in the
current directory. No source files are modified.

---

## How it works

```
Scope → Path discovery → Phase 1: parallel analyzers (one per path) →
Dedup candidates → Phase 2: parallel verifiers (worktree-isolated, one per concern) →
Aggregate VERIFIED only → Markdown + JSON report
```

On Claude Code and Codex CLI, both phases are parallel. On Copilot, both
phases are sequential — same architecture, just slower wall clock.

The Claude Code port also dispatches **2 additional Opus 4.7 holistic
analyzers** in Phase 1 that review the change as a whole (architecture &
consistency, data flow & trust boundaries). They surface candidates only;
verification stays on Sonnet 4.6 for cost.

---

## Caveats (apply to all platforms)

- **Cost is real.** Sonnet 4.6 / GPT-5.5 max-reasoning agents add up fast
  because goobreview spawns one per execution path plus one per concern.
  The skill prints exact agent counts before launching but deliberately
  does not estimate dollar cost — predictions are usually wrong. Watch
  your usage dashboard if you care about spend.
- **Verification asymmetry.** Races, perf regressions, and memory leaks
  tend to come back INCONCLUSIVE rather than VERIFIED. goobreview is
  strongest on logic, null, type, injection, and auth bugs.
- **Heuristic path discovery.** Dynamic dispatch (reflection, plugin
  loaders, eval) can hide entry points. Use `<path>` mode to override.
- **Untested at scale.** Try on a small PR first to surface friction.

---

## File layout

```
goobreview/
├── README.md                      # this file
├── claude-code/
│   ├── SKILL.md                   # orchestration + Agent-tool spawning
│   └── lib/
│       ├── path-discovery.md      # entry-point heuristics per language
│       ├── repro-strategies.md    # verification recipes for 25 bug classes
│       └── severity-rubric.md
├── codex/
│   ├── SKILL.md                   # orchestration + native subagent spawning
│   └── lib/                       # same files as claude-code/lib/
└── github-copilot/
    └── goobreview.agent.md        # single-file custom agent (sequential)
```

The `lib/` content is duplicated between `claude-code/` and `codex/` so each
port is self-contained for installation. The Copilot agent inlines the
condensed bug-class recipes and links back here for the full library.

---

## License

MIT
