---
name: goobreview
description: |
  Verification-first multi-agent code review for Codex CLI. Dispatches one
  high-reasoning subagent per execution path; every candidate concern is
  independently reproduced in an isolated git worktree before it can be
  reported. No false positives: only verified bugs reach the final report.
  Does NOT implement fixes — outputs detailed findings with proposed-fix
  diffs for another agent or a human to apply.

  Three modes: PR diff (default), component path, or whole-codebase scan.

  Use when asked to "goobreview", "deep review", "verified review", or
  "ultrareview locally".
---

# goobreview — for Codex CLI

A local mirror of Anthropic's /ultrareview, ported to Codex CLI's native
multi-agent primitives:
- **One high-reasoning subagent per execution path** (Codex spawns these in
  parallel; default cap is 6 concurrent threads, raise it in config if you
  want full parallelism on large reviews)
- **Mandatory reproduction**: every reported concern is reproduced in an
  isolated git worktree before reporting
- **No fixes applied**: report-only, with proposed-fix diffs

You (the orchestrator) coordinate two phases:
1. **Analyzers** surface candidate concerns from each execution path
2. **Verifiers** reproduce each candidate in an isolated worktree; only
   verified ones are reported

Use the highest-reasoning model available (e.g., `gpt-5.4`). Take your time.
Depth wins over speed.

---

## Step 0 — Resolve scope

Parse the user's invocation:

| Invocation | Mode | Target |
|---|---|---|
| `goobreview` | PR (local) | `git diff <default>...HEAD` + uncommitted |
| `goobreview <N>` (integer) | PR (gh) | `gh pr diff <N>` |
| `goobreview <path>` | Component | All entry points reachable from `<path>` |
| `goobreview --all` | Codebase | All entry points in the repo |

**Always require a git repo.** If `git rev-parse --is-inside-work-tree` fails,
stop with a clear message.

For PR mode (local):
```bash
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo main)
git diff "$DEFAULT_BRANCH"...HEAD --stat
git diff "$DEFAULT_BRANCH"...HEAD --name-only > /tmp/goobreview-changed-files.txt
git diff "$DEFAULT_BRANCH"...HEAD > /tmp/goobreview-diff.patch
```

For PR mode (gh):
```bash
gh pr view "$PR_NUM" --json title,baseRefName,headRefName,files
gh pr diff "$PR_NUM" > /tmp/goobreview-diff.patch
gh pr diff "$PR_NUM" --name-only > /tmp/goobreview-changed-files.txt
```

For `--all`, ALWAYS show cost estimate and confirm before continuing.

---

## Step 1 — Discover execution paths

Read `~/.codex/skills/goobreview/lib/path-discovery.md` for the per-language
heuristics.

An **execution path** = one entry point + its reachable call chain.

Entry-point classes to look for:
- HTTP route handlers (Express, FastAPI, Flask, Django, Rails, Gin, etc.)
- CLI entry points (`main()`, `cmd/*`, `bin/*`, `package.json:bin`)
- Exported top-level functions/classes from package roots
- Queue/event/webhook handlers
- Cron / scheduled tasks
- Public methods on exported services / controllers

For PR mode: keep only paths whose reachable file set intersects the changed
files. Use `grep` / `rg` to walk imports/callers transitively, bounded by
repo root and ignoring vendored dirs.

Write one descriptor per path to `/tmp/goobreview-paths.json`:

```json
{
  "path_id": "P01",
  "entry": "src/api/users.ts:createUser",
  "entry_kind": "http_route",
  "lang": "typescript",
  "reachable_files": ["src/api/users.ts", "src/db/users.ts"],
  "diff_intersection": ["src/db/users.ts"]
}
```

**Cap at 50 paths by default.** Ask the user before exceeding.

---

## Step 2 — Cost & runtime confirmation

Before launching subagents, print:

```
goobreview launching:
  Mode:       <pr|component|codebase>
  Scope:      <one-line summary>
  Paths:      N
  Analyzers:  N subagents (gpt-5.4, max reasoning, parallel up to thread cap)
  Verifiers:  est. 2-5× per path (worktree-isolated, parallel)
  Wall time:  est. <range> min
  Cost:       est. $<range>

If your Codex thread cap is below N, consider raising it
(see `/permissions` or config). Each subagent runs with maximum reasoning
effort. No source files will be modified.
```

For `--all` mode or N > 20 paths, require explicit confirmation.

---

## Step 3 — Phase 1: Spawn path analyzers (parallel)

Codex supports native parallel subagent spawning. Instruct Codex with a
single prompt that asks it to dispatch all N analyzers at once:

> "Spawn N parallel subagents, one for each path in
> `/tmp/goobreview-paths.json`. Each subagent uses the highest-reasoning
> model available with maximum reasoning effort. Each subagent receives the
> ANALYZER PROMPT below with its path's descriptor and the relevant diff
> hunks inlined. Wait for all subagents to return their JSON outputs and
> consolidate them."

Codex's default thread cap is 6. Increase via config if you have > 6 paths
and want true parallelism.

### ANALYZER PROMPT

```
You are reviewing ONE execution path inside a codebase. You are one of N
parallel reviewers; each other path is owned by a different agent. Take
your time. Think deeply. Reason exhaustively.

PATH DESCRIPTOR:
<inline the path JSON>

DIFF (if PR mode, hunks touching this path's reachable_files):
<inline diff>

YOUR JOB:
1. Read every file in reachable_files completely. Then read:
   - The types/interfaces these functions take and return
   - Every caller of the entry symbol (use grep / rg)
   - Every callee that crosses a module boundary
   - Existing tests for this path
   You have unlimited time. Re-read files if you need to.

2. Mentally execute the path with:
   - Realistic happy-path inputs
   - Empty/null/undefined inputs
   - Boundary values (0, -1, MAX_INT, empty string, very long string)
   - Adversarial inputs (SQL/HTML/path traversal payloads where relevant)
   - Concurrent invocations (where relevant)
   - Failure modes of downstream calls

3. Surface CANDIDATE concerns. A candidate is anything that COULD be a real
   bug. Do NOT pre-filter for confidence — verification happens in the next
   phase and will discard false positives. But also do not invent issues
   that have no concrete failure mode.

4. For each candidate, classify the bug class and propose a repro strategy.

BUG CLASSES (pick ONE per candidate, or "other"):
  logic_bug, null_undefined, type_error, race_condition, deadlock,
  sql_injection, xss, ssrf, path_traversal, auth_bypass, data_leak,
  n_plus_one, perf_regression, resource_leak, error_swallowing, api_misuse,
  config_env, dead_code, off_by_one, integer_overflow, money_calculation,
  timezone, regex_dos, supply_chain

DO NOT REPORT:
- Style preferences, naming opinions, "could be more idiomatic"
- Suggestions without a concrete failure mode
- "Consider adding tests" (not a bug)
- "Documentation could be better" (not a bug)

OUTPUT FORMAT — end your message with this JSON block and nothing after it:

```json
{
  "path_id": "P01",
  "candidates": [
    {
      "concern_id": "P01-C01",
      "class": "null_undefined",
      "file": "src/db/users.ts",
      "lines": "47-52",
      "severity_guess": "high",
      "hypothesis": "createUser does not check input.email is non-null before calling .toLowerCase(), crashes with TypeError when client sends {email: null}",
      "repro_strategy": "Call createUser({email: null, name: 'x'}) in a unit test and assert it does not throw TypeError",
      "context_excerpt": "function createUser(input) {\n  const email = input.email.toLowerCase();\n  ...\n}"
    }
  ]
}
```

If you find no candidates after thorough analysis, return:
```json
{"path_id": "P01", "candidates": []}
```

Take your time. The slower and more thorough, the better.
```

When all analyzer subagents return, parse each one's JSON from its final message.

---

## Step 4 — Collect and dedup candidates

Aggregate all candidates into a flat list. Apply dedup:

Two candidates are duplicates if they share `(class, file, line-range overlap ≥ 50%)`.

When merging duplicates:
- Keep the higher `severity_guess` (critical > high > medium > low)
- Concatenate hypotheses if they describe different aspects
- Keep the union of context excerpts

Write merged list to `/tmp/goobreview-candidates.json`.

If 0 candidates: skip to Step 7 with a clean report.
If > 100 candidates: ask the user whether to verify all or sample top-50-by-severity.

---

## Step 5 — Phase 2: Spawn verifiers (parallel, worktree-isolated)

For each unique candidate, spawn a verifier subagent in an isolated git worktree.

Instruct Codex:

> "For each candidate in `/tmp/goobreview-candidates.json`, spawn a parallel
> subagent in its own git worktree. Create the worktree with
> `git worktree add /tmp/goob-<concern_id> HEAD`. Each subagent uses the
> highest-reasoning model with maximum reasoning effort. Each subagent
> receives the VERIFIER PROMPT below with the candidate JSON inlined.
> Wait for all subagents to complete. After collecting results, remove the
> worktrees with `git worktree remove /tmp/goob-<concern_id> --force`."

Codex inherits sandbox policies into subagents, so the worktree isolation is
respected.

### VERIFIER PROMPT

```
You verify whether a candidate concern is a REAL bug by reproducing it in
this isolated git worktree. You are alone on this concern. Take your time.
Think deeply.

CANDIDATE:
<inline candidate JSON>

CONTEXT: You are inside a fresh git worktree at /tmp/goob-<concern_id>.
You can:
- Write any files (test files, repro scripts, configs)
- Run the project's test runner, type checker, linter
- Install dependencies if missing
- Modify any source code
Nothing you do affects the user's main working tree.

YOUR JOB:
1. Read ~/.codex/skills/goobreview/lib/repro-strategies.md and find the
   recipe for class="<class>".

2. Construct a MINIMAL reproduction following the recipe:
   - logic_bug, null_undefined, off_by_one, money_calculation, timezone,
     integer_overflow, error_swallowing
       → write a focused unit test that demonstrates the failure
   - type_error → run the project's type checker; capture the error
   - race_condition, deadlock → write a stress test with N=1000-10000
     concurrent ops; check for inconsistent state or hangs
   - sql_injection → construct a payload (e.g. `' OR 1=1--`), run the code
     path, assert parameterization
   - xss → construct a payload (e.g. `<script>alert(1)</script>`); assert
     it appears escaped in output
   - ssrf, path_traversal → call with internal/escape payloads; assert
     rejection
   - auth_bypass → simulate unauthorized principal; assert rejection
   - n_plus_one → instrument DB driver, count queries; assert bounded
   - perf_regression → benchmark against baseline; assert wall time
   - resource_leak → loop N=10000 times; measure RSS or handles; assert
     not growing linearly
   - dead_code → grep + static analysis; show unreachability
   - config_env → minimal repro project with the misconfiguration
   - api_misuse → run the call, show contract violation
   - regex_dos → run regex with catastrophic input; measure time
   - supply_chain → check lockfile vs advisories

3. RUN your reproduction. Capture the EXACT command and EXACT output.

4. Make a verdict:
   - VERIFIED      The repro demonstrates the bug exists.
   - DISPROVEN     The repro shows the bug does NOT exist.
   - INCONCLUSIVE  You could not construct a reliable repro. Explain why.

RULES:
- VERIFIED requires a concrete, runnable reproduction with captured output.
- Do not return VERIFIED based on inspection alone — you must run something.
- If you cannot install the test framework, return INCONCLUSIVE.

OUTPUT FORMAT — end your message with this JSON block:

```json
{
  "concern_id": "P01-C01",
  "verdict": "VERIFIED",
  "severity_final": "high",
  "title": "createUser crashes on null email",
  "evidence": {
    "repro_files": [
      {
        "path": "test/repro-P01-C01.test.ts",
        "content": "import { createUser } from '../src/db/users';\n\ntest('handles null email', () => {\n  expect(() => createUser({email: null, name: 'x'})).not.toThrow(TypeError);\n});\n"
      }
    ],
    "command": "npx vitest run test/repro-P01-C01.test.ts",
    "output": "FAIL test/repro-P01-C01.test.ts\n  ✗ handles null email\n    TypeError: Cannot read properties of null (reading 'toLowerCase')\n      at createUser (src/db/users.ts:47)\n",
    "exit_code": 1
  },
  "proposed_fix": {
    "file": "src/db/users.ts",
    "diff": "@@ -45,6 +45,9 @@\n function createUser(input) {\n+  if (input.email == null) {\n+    throw new ValidationError('email is required');\n+  }\n   const email = input.email.toLowerCase();"
  },
  "explanation": "Why this is a real bug, what triggers it in production, what the fix does.",
  "notes": "Consider also validating other required fields if schema validation is not done upstream."
}
```

For DISPROVEN, evidence still includes the repro you ran, but its output
shows the test passing or the attack being rejected.

For INCONCLUSIVE, include `"reason": "..."` describing what's missing.

Take your time. A wrong verdict is worse than a slow one.
```

When all verifier subagents return, parse their JSON. Then tear down the
worktrees:

```bash
for id in $(jq -r '.[].concern_id' /tmp/goobreview-candidates.json); do
  git worktree remove "/tmp/goob-$id" --force 2>/dev/null || true
done
```

---

## Step 6 — Aggregate verified findings

Filter verifier results:
- Keep `VERIFIED` for the main findings list
- Keep `DISPROVEN` for the appendix (transparency)
- Keep `INCONCLUSIVE` for the appendix (transparency)

Sort verified findings by:
1. Severity descending (critical > high > medium > low)
2. File path
3. Line number

Read `~/.codex/skills/goobreview/lib/severity-rubric.md` to normalize severity.

---

## Step 7 — Write the report

Two output files, both in the current working directory:
- `./goobreview-report-<ISO-timestamp>.md` — human-readable
- `./goobreview-report-<ISO-timestamp>.json` — machine-readable

Use `date -u +"%Y-%m-%dT%H-%M-%SZ"` for the timestamp.

### Markdown structure

```markdown
# goobreview report

**Generated:** <ISO timestamp>
**Mode:** <pr|component|codebase>
**Scope:** <description>
**Platform:** Codex CLI

## Summary

| Metric | Count |
|---|---|
| Paths analyzed | N |
| Candidates surfaced | M |
| **Verified findings** | **K** |
| Disproven (false positives) | F |
| Inconclusive | I |
| Wall clock | XXm |
| Analyzer subagents | N |
| Verifier subagents | V |

### Severity breakdown
- Critical: X
- High: Y
- Medium: Z
- Low: W

---

## Verified findings

### 1. [CRITICAL] <title>
**File:** `path/to/file.ext:47-52`
**Class:** `null_undefined`
**Path:** P01 (POST /api/users)

**Explanation:**
<verifier explanation>

**Reproduction (verified):**
```
$ <command>
<output>
```

<details>
<summary>Reproduction file contents</summary>

```<lang>
<contents>
```
</details>

**Proposed fix:**
```diff
<diff>
```

**Notes:** <verifier notes>

---

(repeat for each verified finding)

---

## Disproven candidates (F)
(transparency)

## Inconclusive candidates (I)
(transparency)

## Paths reviewed
(P01 ... PN summary)

---

## How to act on this report

Copy the diff from finding #N and apply it, or hand it to another agent.
```

### JSON structure

```json
{
  "version": 1,
  "platform": "codex",
  "generated_at": "2026-05-13T14:33:00Z",
  "mode": "pr",
  "scope": "diff vs origin/main, 7 files",
  "stats": {
    "paths": 5,
    "candidates_raw": 14,
    "candidates_unique": 11,
    "verified": 4,
    "disproven": 5,
    "inconclusive": 2,
    "analyzer_subagents": 5,
    "verifier_subagents": 11,
    "wall_clock_seconds": 1843
  },
  "severity_breakdown": {"critical": 1, "high": 2, "medium": 1, "low": 0},
  "findings": [<verified records>],
  "disproven": [<disproven records>],
  "inconclusive": [<inconclusive records>],
  "paths": [<path descriptors with candidate counts>]
}
```

---

## Step 8 — Final user message

Print to the user (terse):
- Path to the markdown report
- Headline counts (`K verified findings: X critical, Y high, Z medium, W low`)
- Cost & time spent
- One-line on next steps

**Do NOT** apply any fixes. **Do NOT** modify any source files. Report only.

---

## Failure modes & recovery

| Failure | Recovery |
|---|---|
| Analyzer returns invalid JSON | Re-prompt the subagent once asking for JSON only. If still bad, drop that path with a note. |
| Worktree creation fails | Fall back to `cp -r` into `/tmp/goobreview-<concern_id>/` and run there. |
| Verifier cannot install deps | Try `--offline` / cached install. If still fails, return INCONCLUSIVE. |
| Test runner not detected | INCONCLUSIVE with reason "no test runner found". |
| > 100 candidates after dedup | Ask the user to verify all or sample top-50-by-severity. |
| 0 paths discovered | Stop with a clear message. |
| Not in a git repo | Stop. |
| Hitting Codex thread cap | Process in batches; total time scales with batch count. |

---

## Internal invariants

- The orchestrator NEVER edits source files in the user's working tree.
- Verifier subagents can edit files, but only in their own worktree.
- Analyzer subagents are read-only.
- Every reported finding has captured `evidence.command` and `evidence.output`.
- The report file is the only thing written to the user's repo.

---

## Done

When the report is written, you're done. The next step belongs to the user
or another agent.
