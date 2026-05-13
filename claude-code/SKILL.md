---
name: goobreview
description: |
  Verification-first multi-agent code review. Dispatches one Sonnet 4.6 agent per
  execution path PLUS 2 Opus 4.7 max-reasoning agents that review the change
  holistically (architecture & consistency, data flow & trust boundaries). Every
  candidate concern is independently reproduced by a Sonnet 4.6 agent in an
  isolated git worktree before it can be reported. No false positives: only
  verified bugs reach the final report. Does NOT implement fixes — outputs
  detailed findings with proposed-fix diffs for another agent or a human to apply.

  Three modes: PR diff (default), component path, or whole-codebase scan.

  Use when asked to "goobreview", "deep review", "verified review", "forensic review",
  "ultrareview locally", or "verify every concern before reporting".
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - Write
triggers:
  - goobreview
  - verified review
  - forensic review
  - deep code review
  - ultrareview locally
---

# goobreview — verification-first multi-agent code review

A local mirror of Anthropic's /ultrareview with stronger guarantees:
- **One Sonnet 4.6 agent per execution path** + **2 Opus 4.7 holistic analyzers** (architecture/consistency + data-flow/trust-boundaries) — not a fixed fleet
- **Mandatory reproduction**: every reported concern was reproduced in an isolated git worktree
- **No fixes applied**: report-only, with proposed-fix diffs ready for another agent

You (the orchestrator) coordinate two phases of spawned agents:
1. **Analyzers** surface candidate concerns from each execution path
2. **Verifiers** reproduce each candidate in isolation; only verified ones are reported

Take your sweet time. Depth wins over speed.

---

## Step 0 — Resolve scope

Parse the user's invocation:

| Invocation | Mode | Target |
|---|---|---|
| `/goobreview` | PR (local) | `git diff <default>...HEAD` + uncommitted |
| `/goobreview <N>` (integer) | PR (gh) | `gh pr diff <N>` |
| `/goobreview <path>` | Component | All entry points reachable from `<path>` |
| `/goobreview --all` | Codebase | All entry points in the repo |

**Always require a git repo.** If `git rev-parse --is-inside-work-tree` fails, stop with a clear message.

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

For `--all`, ALWAYS confirm scope via AskUserQuestion before continuing.

Print a one-line scope summary the user can sanity-check before launching agents.

---

## Step 1 — Discover execution paths

Read `lib/path-discovery.md` for the full per-language heuristics. The short version:

An **execution path** = one entry point + its reachable call chain.

Entry-point classes to look for:
- HTTP route handlers (Express `app.get`, FastAPI `@app.get`, Flask `@app.route`, Django `urls.py`, Rails `routes.rb`, Gin/Echo/Fiber handlers, ASP.NET `[Route]`, etc.)
- CLI entry points (`main()`, `cmd/*`, `bin/*`, `package.json:bin`, `setup.py:entry_points`, `pyproject.toml:scripts`)
- Exported top-level functions/classes from package roots (`index.*`, `mod.rs`, `__init__.py`, `pkg/*`)
- Queue/event/webhook handlers (`@on_message`, `@app.task`, SQS/Kafka consumers)
- Cron / scheduled tasks
- Public methods on exported services / controllers
- React/Vue/Svelte page components (only if reviewing frontend; ignore otherwise)

**For PR mode:** keep only paths whose reachable file set intersects the changed files. Use Grep to walk imports/callers transitively, bounded by repo root and ignoring vendored dirs (`node_modules/`, `.venv/`, `vendor/`, `target/`, `dist/`, `build/`).

**For Component mode:** all entry points whose reachable set touches `<path>`.

**For --all:** every entry point you can find.

Write one descriptor per path to `/tmp/goobreview-paths.json`:

```json
{
  "path_id": "P01",
  "entry": "src/api/users.ts:createUser",
  "entry_kind": "http_route",
  "lang": "typescript",
  "reachable_files": ["src/api/users.ts", "src/db/users.ts", "src/validators/user.ts"],
  "diff_intersection": ["src/db/users.ts"]
}
```

**Cap at 50 paths by default.** If more, ask the user to narrow scope or confirm proceeding.

If zero paths discovered, stop and tell the user — there's nothing to review.

---

## Step 2 — Launch confirmation

Before launching analyzers, print a deterministic launch summary:

```
goobreview launching:
  Mode:       <pr|component|codebase>
  Scope:      <one-line summary>
  Paths:      N
  Path analyzers:     N Sonnet 4.6 agents (max reasoning, parallel)
  Holistic analyzers: 2 Opus 4.7 agents (max reasoning, parallel)
  Verifiers:          spawned per concern after Phase 1 dedup (Sonnet 4.6, worktree-isolated, parallel)

Each agent runs with `ultrathink` to maximize reasoning depth. This is by design.
No source files will be modified. The report writes to ./goobreview-report-*.md.
```

**Do NOT estimate dollar cost or wall-clock time.** The agent counts are
deterministic and useful; cost and time predictions are guesses that are
usually wrong and alarm the user. If the user wants budget awareness, they
should watch their usage dashboard.

For `--all` mode or N > 20 paths, REQUIRE explicit confirmation via
AskUserQuestion describing scope and agent count only — no cost or time
estimate. For smaller scopes, just print and proceed.

---

## Step 3 — Phase 1: Spawn path analyzers (parallel)

Dispatch ONE Agent call per path, **all in a single message** so they run concurrently.

Each Agent call:
- `subagent_type: "general-purpose"`
- `model: "sonnet"`
- `description: "Analyze path P0X"`
- `prompt`: see ANALYZER PROMPT below, with the path descriptor and any relevant diff hunks inlined

### ANALYZER PROMPT

```
You are reviewing ONE execution path inside a codebase. You are one of N parallel
reviewers; each other path is owned by a different agent. Take your time. Be thorough.

**ultrathink**

PATH DESCRIPTOR:
<inline the path JSON>

DIFF (if PR mode, the hunks touching this path's reachable_files):
<inline diff>

YOUR JOB:
1. Read every file in reachable_files completely. Then read:
   - The types/interfaces these functions take and return
   - Every caller of the entry symbol (use Grep)
   - Every callee that crosses a module boundary
   - Existing tests for this path (search for the entry symbol in test files)
   You have unlimited time. Re-read files if you need to.

2. Mentally execute the path with:
   - Realistic happy-path inputs
   - Empty/null/undefined inputs
   - Boundary values (0, -1, MAX_INT, empty string, very long string)
   - Adversarial inputs (SQL/HTML/path traversal payloads where relevant)
   - Concurrent invocations (where relevant)
   - Failure modes of downstream calls (DB down, network timeout, etc.)

3. Surface CANDIDATE concerns. A candidate is anything that COULD be a real bug.
   Do NOT pre-filter for confidence — verification happens in the next phase
   and will discard false positives. It is better to surface and have something
   disproven than to miss a real bug. But also do not invent issues that have
   no concrete failure mode.

4. For each candidate, classify the bug class and propose a repro strategy.

BUG CLASSES (pick ONE per candidate, or "other"):
  logic_bug          wrong output for valid input
  null_undefined     null/undefined dereference crashes
  type_error         type mismatch caught at compile or runtime
  race_condition     concurrent access produces wrong state
  deadlock           concurrent code can hang
  sql_injection      unparameterized SQL with user input
  xss                unescaped HTML/JS in output
  ssrf               server-side request forgery
  path_traversal     unsanitized path used in filesystem op
  auth_bypass        missing/weak authorization check
  data_leak          sensitive data in response/log/error
  n_plus_one         query in loop or eager-load miss
  perf_regression    complexity worse than necessary
  resource_leak      file/conn/memory not released
  error_swallowing   exception caught and ignored
  api_misuse         wrong contract with library/API
  config_env         missing env var, bad default
  dead_code          code that cannot be reached
  off_by_one         index/range arithmetic wrong
  integer_overflow   arithmetic overflow / precision loss
  money_calculation  float math on currency
  timezone           datetime without TZ awareness
  regex_dos          catastrophic backtracking
  supply_chain       dependency with known vuln

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

### Phase 1b — 2 holistic Opus analyzers (parallel with Phase 1a)

In parallel with the per-path Sonnet analyzers, dispatch **2 Opus 4.7
max-reasoning analyzers** that review the change as a whole. They catch
cross-cutting issues that path-scoped reviewers miss: inconsistent
refactors, invariant violations spanning files, data flow across trust
boundaries, API contract drift between caller and callee paths.

These analyzers SURFACE candidates only — they do NOT verify. Their
candidates go through the same dedup + Sonnet verification pipeline as
path-analyzer candidates. The expensive Opus model is used here because
holistic reasoning is where it pays off; verification is mechanical and
stays on Sonnet.

Two lenses, one Agent each. Spawn both in the same message as the path
analyzers so all of Phase 1 runs in parallel:

- **H1 — Architecture & consistency**: cross-file invariants, refactor
  completeness, partial migrations, stale callers, dependency/module
  boundary issues, API contract drift between caller and callee
- **H2 — Data flow & trust boundaries**: user input flowing across paths
  into sinks (SQL, shell, FS, HTTP, render), auth boundaries broken by
  the change, sensitive data exposure across the request lifecycle,
  state mutations crossing transaction boundaries

Each Agent call:
- `subagent_type: "general-purpose"`
- `model: "opus"`  ← deliberately heavier than the Sonnet path analyzers; only 2 of these so cost is bounded
- `description: "Holistic analyzer H1"` (or H2)
- `prompt`: see HOLISTIC ANALYZER PROMPT below with the appropriate LENS

#### HOLISTIC ANALYZER PROMPT

```
You are a holistic reviewer. You read the change as a whole — not through
any single execution path — and surface candidate concerns that require
cross-cutting visibility to spot. Take your time. Reason exhaustively.

**ultrathink**

LENS: <inject "H1: Architecture & consistency" OR "H2: Data flow & trust boundaries">

SCOPE:
<inject the full diff for PR mode, or the full file list for component / codebase mode>

CHANGED FILES:
<inject list of files changed>

YOUR JOB:
1. Read enough of the codebase to form a holistic view. Read shared
   types, interfaces, schema and config files, and any module referenced
   from multiple changed files. You have unlimited time.

2. Reason about the change as a whole:
   - **LENS H1** — does the change leave the codebase in a consistent
     state? Do callers match callees? Do related files agree on field
     names, types, and contracts? Is the refactor complete or did it
     leave stale callers, partial migrations, or dead branches?
   - **LENS H2** — trace user-controlled data through the changed code.
     Where does it cross trust boundaries? Where does it reach sinks
     (SQL, shell, FS, HTTP, eval, render, log)? Are auth checks
     consistent across all entry paths to the same privileged operation?

3. Surface candidate concerns that ONLY a holistic view would catch.
   Do NOT duplicate single-function bugs a path-scoped analyzer already
   sees (those run in parallel with you). Focus on issues that require
   seeing multiple files or the diff as a whole.

4. Use the SAME bug classes as path analyzers. Set `path_id` to `"H1"`
   or `"H2"` so dedup knows where the candidate came from. Do not return
   a verdict — verification happens in the next phase.

OUTPUT FORMAT — same JSON shape as path analyzers:

```json
{
  "path_id": "H1",
  "candidates": [
    {
      "concern_id": "H1-C01",
      "class": "api_misuse",
      "file": "src/jobs/seed.js",
      "lines": "42-48",
      "severity_guess": "high",
      "hypothesis": "createUser now requires tenantId (per src/db/users.ts:47 change), but seed.js still calls it without one. TS compiler catches updated TS sites but the dynamic call in seed.js will fail at runtime.",
      "repro_strategy": "Run `node scripts/seed.js` against the new createUser signature; expect a ValidationError",
      "context_excerpt": "createUser({ email: 'admin@x', name: 'Admin' });"
    }
  ]
}
```

If you find no candidates after thorough analysis, return:
```json
{"path_id": "H1", "candidates": []}
```

Take your time. Cross-cutting bugs are the most expensive to ship.
```

When all analyzer agents return (both path and holistic), parse each one's JSON.

---

## Step 4 — Collect and dedup candidates

Aggregate all candidates into a flat list. Apply dedup:

Two candidates are duplicates if they share `(class, file, line-range overlap ≥ 50%)`.

When merging duplicates:
- Keep the higher `severity_guess` (critical > high > medium > low)
- Concatenate hypotheses if they describe different aspects
- Keep the union of context excerpts

Write merged list to `/tmp/goobreview-candidates.json`.

If 0 candidates after dedup: skip to Step 7 with a clean report.
If > 100 candidates: warn the user and ask whether to verify all or sample top-50-by-severity.

### Announce candidates (print to user)

Before launching verifiers, print the full candidate list to the user as a
numbered, one-line-per-item summary. Sort by `severity_guess` descending
(critical > high > medium > low), then by file, then by line. Render as
markdown so it shows up as a nicely formatted list:

```
Phase 1 complete — N candidate concerns surfaced:

1. **[CRIT]** `sql_injection` — `src/db/orders.ts:33-41` — orderSearch concatenates user query
2. **[HIGH]** `null_undefined` — `src/db/users.ts:47-52` — createUser crashes on null email
3. **[MED]**  `n_plus_one` — `src/api/users.ts:120-145` — listUsers issues N queries per call
4. **[LOW]**  `dead_code` — `src/utils/legacy.ts:88` — formatLegacyDate has no callers

Launching N verifiers (Sonnet 4.6 max reasoning, worktree-isolated)...
```

Format rules:
- Severity tag uppercased and abbreviated: `[CRIT]`, `[HIGH]`, `[MED]`, `[LOW]`
- Class in backticks, snake_case
- `file:lines` in backticks
- Hypothesis is the short one-line description (truncate at ~80 chars if longer)
- One line per candidate (truncate hypothesis rather than wrapping)
- Use a markdown numbered list — Claude Code's renderer makes this look clean

This list is printed to the user inline, not written to a file. The user
should be able to see at a glance what's about to be verified.

---

## Step 5 — Phase 2: Spawn verifiers (parallel, worktree-isolated)

For each unique candidate, spawn an Sonnet 4.6 max-reasoning verifier **with worktree isolation**.

Each Agent call:
- `subagent_type: "general-purpose"`
- `model: "sonnet"`
- `isolation: "worktree"`  ← critical: gives the verifier a fresh worktree
- `description: "Verify P0X-C0Y"`
- `prompt`: see VERIFIER PROMPT below

Batch all verifier calls into one or more parallel messages (batch size of ~10 is fine — the harness handles parallelism). If you have 50 candidates, that's 5 batched messages.

### VERIFIER PROMPT

```
You verify whether a candidate concern is a REAL bug by reproducing it in this
isolated git worktree. You are alone on this concern. Take your time.

**ultrathink**

CANDIDATE:
<inline candidate JSON>

CONTEXT: You are inside a fresh git worktree of the user's repo. You can:
- Write any files (test files, repro scripts, configs)
- Run the project's test runner, type checker, linter
- Install dependencies if missing
- Modify any source code
Nothing you do affects the user's main working tree. The worktree auto-cleans
when you finish.

YOUR JOB:
1. Read lib/repro-strategies.md (in the goobreview skill directory) for the
   recipe for class="<class>". The file is at ~/.claude/skills/goobreview/lib/
   repro-strategies.md — use Read directly.

2. Construct a MINIMAL reproduction following the recipe:
   - logic_bug, null_undefined, off_by_one, money_calculation, timezone,
     integer_overflow, error_swallowing
       → write a focused unit test that demonstrates the failure
   - type_error
       → run the project's type checker (tsc, mypy, go vet, cargo check, etc.)
         on the relevant files; capture the error
   - race_condition, deadlock
       → write a stress test with N=1000-10000 concurrent ops; check for
         inconsistent state or hangs (use timeout)
   - sql_injection
       → construct a payload (e.g. `' OR 1=1--`), run the code path with it,
         either capture the resulting SQL (if testable) or actually execute
         against a test DB; assert parameterization
   - xss
       → render output with a payload like `<script>alert(1)</script>`; assert
         it appears escaped
   - ssrf
       → call the path with an internal-only URL (127.0.0.1, 169.254.169.254);
         assert it is rejected
   - path_traversal
       → call with `../../etc/passwd` style input; assert it is rejected
   - auth_bypass
       → simulate a request from an unauthorized principal; assert it is
         rejected
   - n_plus_one
       → instrument the DB driver to count queries; run the path; assert
         query count is bounded
   - perf_regression
       → write a benchmark against a reasonable baseline; assert wall time
   - resource_leak
       → loop the operation N=10000 times; measure RSS or open-handle count;
         assert it does not grow linearly
   - dead_code
       → grep + static analysis; show the code is unreachable
   - config_env
       → construct minimal repro project; show the misconfiguration causes
         the failure
   - api_misuse
       → run the call (or a mock) and show the contract violation
   - regex_dos
       → run the regex with a known catastrophic input; measure time
   - supply_chain
       → check lockfile / advisory database; show the vuln applies

3. RUN your reproduction. Capture the EXACT command and EXACT output.

4. Make a verdict:
   - VERIFIED      The repro demonstrates the bug exists.
   - DISPROVEN     The repro shows the bug does NOT exist (e.g., your failing
                   test actually passes because the code handles the case).
   - INCONCLUSIVE  You could not construct a reliable repro. Explain precisely
                   what's missing.

RULES:
- VERIFIED requires a concrete, runnable reproduction with captured output.
- DISPROVEN requires showing what you ran and why it disproved the hypothesis.
- Do not return VERIFIED based on inspection alone — you must run something.
- For class=dead_code, "running" means running the static analysis (grep,
  callgraph) that proves unreachability.
- If you cannot install the test framework or run tests in the worktree,
  return INCONCLUSIVE and say why.

OUTPUT FORMAT — end your message with this JSON block and nothing after it:

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
  "notes": "Consider also validating `name` if schema validation is not done upstream."
}
```

For DISPROVEN, the `evidence` field still includes the repro you ran, but its
output shows the test passing or the attack being rejected.

For INCONCLUSIVE, include `"reason": "..."` describing what's missing.

Take your time. A wrong verdict is worse than a slow one.
```

When all verifier agents return, parse their JSON.

### Announce verdicts (print to user)

After all verifiers return, print the same list a second time — same
numbering, same order — but with DISPROVEN and INCONCLUSIVE rows struck
through, and a verdict tag appended to every row.

```
Verification complete — K of N candidates verified as real bugs:

1. **[CRIT]** `sql_injection` — `src/db/orders.ts:33-41` — orderSearch concatenates user query  ✓ VERIFIED
2. **[HIGH]** `null_undefined` — `src/db/users.ts:47-52` — createUser crashes on null email  ✓ VERIFIED
3. ~~**[MED]**  `n_plus_one` — `src/api/users.ts:120-145` — listUsers issues N queries per call~~  ✗ DISPROVEN — eager:true is already used at line 130
4. ~~**[LOW]**  `dead_code` — `src/utils/legacy.ts:88` — formatLegacyDate has no callers~~  ? INCONCLUSIVE — symbol is referenced via dynamic dispatch
```

Format rules:
- Same item order as the Phase 1 announcement (so the user can compare line-by-line)
- `~~...~~` strikethrough wraps the descriptive part (severity, class, file, hypothesis), NOT the verdict tag
- Verdict tag appended at end of each row:
  - `✓ VERIFIED`
  - `✗ DISPROVEN — <one-line reason>`
  - `? INCONCLUSIVE — <one-line reason>`
- One line per candidate

This is printed to the user inline. The report file (Step 7) contains the
full VERIFIED entries with evidence and proposed fixes, plus appendices
for the others.

---

## Step 6 — Aggregate verified findings

Filter verifier results:
- Keep `VERIFIED` for the main findings list
- Keep `DISPROVEN` for the "Disproven candidates" appendix (transparency)
- Keep `INCONCLUSIVE` for the "Inconclusive candidates" appendix (transparency)

Sort verified findings by:
1. Severity descending (critical > high > medium > low)
2. File path
3. Line number

Read `lib/severity-rubric.md` to confirm severity normalization (verifiers may
have used slightly different scales; map them to the canonical four levels).

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

## Summary

| Metric | Count |
|---|---|
| Paths analyzed | N |
| Candidates surfaced | M |
| **Verified findings** | **K** |
| Disproven (false positives) | F |
| Inconclusive | I |
| Wall clock | XXm |
| Analyzer agents | N |
| Verifier agents | V |

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

These were surfaced by an analyzer but the verifier proved them wrong. Listed
here for transparency.

- [P01-C03] **null_undefined** in `src/foo.ts:120` — analyzer thought X but
  verifier showed Y. Repro: `<command>` returned `<output>`.

(repeat)

---

## Inconclusive candidates (I)

Verifier could not construct a reliable reproduction. Surface for human triage.

- [P02-C01] **race_condition** in `src/queue.go:88` — could not trigger reliably
  with N=10000 iterations. Would need: `<what's missing>`.

(repeat)

---

## Paths reviewed

- P01: `src/api/users.ts:createUser` (http_route) — 2 candidates surfaced, 1 verified
- P02: `src/queue/worker.go:processJob` (queue_handler) — 4 candidates surfaced, 2 verified
- ...

---

## How to act on this report

To apply a proposed fix:
- Copy the diff from finding #N and apply it manually, OR
- Run a follow-up agent: `claude -p "apply finding #N from goobreview-report-<ts>.md"`

To re-verify after fixing:
- Re-run `/goobreview` with the same scope.
```

### JSON structure

```json
{
  "version": 1,
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
    "analyzer_agents": 5,
    "verifier_agents": 11,
    "wall_clock_seconds": 1843
  },
  "severity_breakdown": {"critical": 1, "high": 2, "medium": 1, "low": 0},
  "findings": [<verified records as emitted by verifiers>],
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
- Wall clock spent (rough)
- One-line on next steps (apply diffs from the report; re-run to re-verify)

**Do NOT** apply any fixes. **Do NOT** modify any source files. Report only.

---

## Failure modes & recovery

| Failure | Recovery |
|---|---|
| Analyzer returns invalid JSON | Re-prompt the agent once asking for JSON only. If still bad, drop that path with a note. |
| Verifier worktree fails to create | Fall back to `/tmp/goobreview-<concern_id>/` rsync copy. |
| Verifier cannot install deps | Try `--offline` / cached install. If still fails, return INCONCLUSIVE with the install error. |
| Test runner not detected in repo | INCONCLUSIVE for that concern with reason "no test runner found". |
| > 100 candidates after dedup | AskUserQuestion: verify all (expensive) or top-50-by-severity. |
| 0 paths discovered | Stop with a clear message: "No execution paths in scope. Check the path argument or PR diff." |
| Not in a git repo | Stop. goobreview requires git. |
| `--all` without confirmation | AskUserQuestion to confirm scope. |

---

## Internal invariants

- The orchestrator (you) NEVER edits source files in the user's working tree.
- Verifiers can edit files, but only in their own worktree.
- Analyzers are read-only.
- Every reported finding has captured `evidence.command` and `evidence.output`.
- The report file is the only thing written to the user's repo.

---

## Done

When the report is written, you're done. The next step belongs to the user or
another agent.
