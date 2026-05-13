# Reproduction strategies by bug class

A verifier reads this file (the orchestrator instructs it to) and follows the
recipe for the candidate's class.

Each recipe gives:
- **Goal** of the reproduction
- **Method** — what to write and run
- **Verdict criteria** — when to return VERIFIED vs DISPROVEN vs INCONCLUSIVE
- **Failure modes** — common reasons the verification can't be done

The verifier is in an isolated git worktree and has full Bash + Write access.

---

## logic_bug

**Goal:** Show the function produces wrong output for some valid input.

**Method:**
1. Identify the function and its expected contract from the hypothesis.
2. Write a unit test in the project's test framework that calls the function
   with the input class described in the hypothesis and asserts the correct
   output.
3. Run the test. If it fails, the bug is verified.

**Verdict:**
- VERIFIED: test fails, output diverges from contract as described.
- DISPROVEN: test passes, the function handles the input correctly.
- INCONCLUSIVE: cannot determine the correct output (ambiguous contract).

**Failure modes:** No test runner in the project. Try `pytest`, `jest`,
`vitest`, `go test`, `cargo test`, `mvn test`. If none work, INCONCLUSIVE.

---

## null_undefined

**Goal:** Show a null or undefined value reaches a dereference and crashes.

**Method:**
1. Write a test that calls the entry point with the null-bearing input from
   the hypothesis.
2. Run it. Capture stack trace if it crashes.

**Verdict:**
- VERIFIED: crash with `TypeError`, `NullPointerException`, `nil dereference`,
  etc. at the line indicated.
- DISPROVEN: no crash; the code handles null gracefully.
- INCONCLUSIVE: input flow is complex enough that you can't construct the call.

---

## type_error

**Goal:** Show the project's type checker catches a real type mismatch.

**Method:**
Run the language-appropriate checker on the file or whole project:
- TypeScript: `npx tsc --noEmit` (or project's typecheck script)
- Python: `mypy <file>` or `pyright <file>`
- Go: `go vet ./...`
- Rust: `cargo check`
- Java: `mvn compile` / `gradle compileJava`

**Verdict:**
- VERIFIED: checker reports error at the indicated location.
- DISPROVEN: checker reports no error.
- INCONCLUSIVE: checker not configured; if you can install it cheaply, do so.

---

## race_condition

**Goal:** Show concurrent access produces inconsistent state.

**Method:**
1. Write a stress test:
   - 1000–10000 iterations of concurrent calls to the code path
   - Use the language's concurrency primitive (goroutines, threads, asyncio,
     workers, etc.)
   - Assert a consistency invariant after each iteration or at the end
2. Run with `timeout 60s` to avoid hangs.

**Verdict:**
- VERIFIED: at least one iteration violates the invariant. Capture the output.
- DISPROVEN: 10000 iterations pass with no violations.
- INCONCLUSIVE: cannot construct the concurrent harness, or violations don't
  reproduce within the time budget. Note that races are inherently flaky;
  prefer to err toward INCONCLUSIVE if N=10000 passes cleanly.

**Note:** Use atomics or locks ONLY to detect races, never to fix them in the
repro (that would hide the bug).

---

## deadlock

**Goal:** Show concurrent calls can hang.

**Method:**
1. Write a test that invokes the code path with the access pattern described
   in the hypothesis (e.g., two threads acquiring locks in different orders).
2. Run with `timeout 30s`.

**Verdict:**
- VERIFIED: test times out at the expected point.
- DISPROVEN: test completes within timeout.
- INCONCLUSIVE: can't construct the pattern reliably.

---

## sql_injection

**Goal:** Show user-controlled input flows into SQL without parameterization.

**Method:**
1. Locate the SQL query construction site identified in the hypothesis.
2. Either:
   - **Static**: read the code carefully. If the query is built with string
     concatenation or template literals containing user input, that's the
     defect. Show the exact line.
   - **Runtime** (preferred when possible): write a test that calls the entry
     point with payload `' OR 1=1--` or `'; DROP TABLE x;--`. Mock or wrap
     the DB driver to capture the final SQL string. Assert that the user
     input appears as a parameter placeholder, not inlined.

**Verdict:**
- VERIFIED: SQL string contains the raw payload (not parameterized).
- DISPROVEN: SQL uses `?` / `$1` / named params and the input is a value.
- INCONCLUSIVE: query construction is too dynamic to verify without a DB.

---

## xss

**Goal:** Show user input reaches output without escaping.

**Method:**
1. Identify the output path (template, JSON response, HTML render).
2. Construct a payload like `<script>alert(1)</script>` or
   `"><img src=x onerror=alert(1)>`.
3. Render the output via the function or template and inspect the result.

**Verdict:**
- VERIFIED: payload appears verbatim in output (HTML-unescaped).
- DISPROVEN: payload appears escaped (`&lt;script&gt;...`) or is rejected.
- INCONCLUSIVE: rendering pipeline is too complex to invoke directly.

---

## ssrf

**Goal:** Show server-side request forgery is possible.

**Method:**
1. Write a test calling the entry point with internal URLs:
   `http://127.0.0.1`, `http://169.254.169.254` (AWS metadata),
   `file:///etc/passwd`, `gopher://`.
2. Use a mock HTTP client or instrument the call to capture the URL that
   would have been requested.

**Verdict:**
- VERIFIED: any of the payloads is accepted and would be fetched.
- DISPROVEN: all payloads are rejected at validation.
- INCONCLUSIVE: cannot instrument the HTTP layer.

---

## path_traversal

**Goal:** Show `../` or absolute paths escape the intended directory.

**Method:**
1. Write a test calling the path with inputs like `../../etc/passwd`,
   `/etc/passwd`, `....//etc/passwd`.
2. Capture the final path that would be read/written.

**Verdict:**
- VERIFIED: resolved path is outside the intended root.
- DISPROVEN: input is rejected or normalized to a safe path.
- INCONCLUSIVE: path resolution flow is unclear.

---

## auth_bypass

**Goal:** Show a privileged action is reachable without proper authorization.

**Method:**
1. Identify the auth check in the path (or its absence).
2. Write a test that simulates an unauthenticated or under-privileged
   principal calling the endpoint:
   - For HTTP: build the request with no/forged credentials.
   - For internal calls: instantiate the call with a low-privilege identity.
3. Assert it is rejected (401, 403, or domain-specific denial).

**Verdict:**
- VERIFIED: call succeeds when it should have been rejected.
- DISPROVEN: call is properly rejected.
- INCONCLUSIVE: auth flow depends on infra you can't simulate in the worktree.

---

## data_leak

**Goal:** Show sensitive data (passwords, tokens, PII) appears in a response,
log, or error.

**Method:**
1. Identify the data field and the output channel from the hypothesis.
2. Write a test that triggers the path and captures the output (response
   body, stderr, log file).
3. Search the captured output for the sensitive field.

**Verdict:**
- VERIFIED: field appears in output unredacted.
- DISPROVEN: field is absent or redacted.
- INCONCLUSIVE: cannot capture the relevant output channel.

---

## n_plus_one

**Goal:** Show the path issues O(N) queries when O(1) or O(log N) is correct.

**Method:**
1. Instrument the DB driver to count or log queries. Use the project's ORM
   instrumentation hook if available (Sequelize logging, Django connection
   queries, SQLAlchemy events, GORM callbacks).
2. Write a test that seeds N=100 records, calls the path, and counts queries.
3. Assert query count is small (e.g., ≤ 5) and constant w.r.t. N.

**Verdict:**
- VERIFIED: query count grows with N (e.g., 100+ for N=100).
- DISPROVEN: query count is bounded.
- INCONCLUSIVE: can't instrument the DB driver in the worktree.

---

## perf_regression

**Goal:** Show algorithmic complexity is worse than necessary.

**Method:**
1. Write a benchmark that runs the path with increasing input sizes
   (e.g., N=100, 1000, 10000).
2. Measure wall time per N.
3. Compare growth: is it O(N), O(N²), O(2^N)?

**Verdict:**
- VERIFIED: growth matches the bad complexity in the hypothesis.
- DISPROVEN: growth is acceptable.
- INCONCLUSIVE: bench would need infra you can't set up.

---

## resource_leak

**Goal:** Show a resource (file handle, connection, memory) grows without
bound.

**Method:**
1. Write a test that calls the path N=10000 times in a loop.
2. Measure resource usage:
   - File handles: `lsof -p $$ | wc -l` before and after
   - Memory: `/proc/$$/status` RSS before and after
   - Connections: pool stats
3. Assert resource usage does not grow linearly with N.

**Verdict:**
- VERIFIED: resource grows ≥ N% with iteration count.
- DISPROVEN: stable.
- INCONCLUSIVE: can't measure the resource type.

---

## error_swallowing

**Goal:** Show an exception is caught and silently dropped.

**Method:**
1. Find the catch block. Inject a controlled exception in the try-body via
   monkey-patching or mocking.
2. Run the code. Check whether the exception was logged or re-raised.

**Verdict:**
- VERIFIED: exception is caught and not surfaced anywhere.
- DISPROVEN: exception is logged / re-raised / handled meaningfully.
- INCONCLUSIVE: can't inject the exception cleanly.

---

## api_misuse

**Goal:** Show the code uses an external API contrary to its documented contract.

**Method:**
1. Read the library's documentation (use WebFetch on its docs page if you
   have internet).
2. Identify the contract violation.
3. Either:
   - Write a test that runs the call and observes the contract violation, OR
   - Use static evidence (the contract is documented, the call doesn't
     match) with a clear citation.

**Verdict:**
- VERIFIED: with either runtime test failure or documented + observed mismatch.
- DISPROVEN: usage is correct.
- INCONCLUSIVE: contract is ambiguous or undocumented.

---

## config_env

**Goal:** Show a config/env issue causes the failure described.

**Method:**
1. Reproduce the misconfiguration in the worktree (set the env var, edit the
   config file, etc.).
2. Run the affected path or startup.
3. Capture the error.

**Verdict:**
- VERIFIED: misconfiguration produces the predicted failure.
- DISPROVEN: configuration is handled correctly.
- INCONCLUSIVE: can't simulate the environment.

---

## dead_code

**Goal:** Show the code is unreachable.

**Method:**
1. Grep for callers of the symbol/file across the repo. Check imports.
2. Check dynamic dispatch sites (reflection, string lookups) if relevant.
3. If the code is in a library, check whether the public API exposes it.

**Verdict:**
- VERIFIED: no callers, no dynamic references, not exported as public API.
- DISPROVEN: at least one caller exists.
- INCONCLUSIVE: dynamic dispatch makes static analysis incomplete.

---

## duplicate_code

**Goal:** Confirm the code is duplicated — either within the changed
files, or reimplements logic that already exists in the codebase.

**Method:**
1. From the candidate's hypothesis, identify the suspected duplicate.
   The analyzer should have given one of:
   - "this block at file A:lines is also at file B:lines" (within-change
     duplication), OR
   - "this new function at file X:lines does what existing util Y at
     file Z:lines already does" (reimplementation of existing code).
2. Confirm the duplication:
   - **Exact / near-exact match**: `diff` the two ranges. Allow
     whitespace and identifier renames; reject only if structural
     similarity is < 80%.
   - **Semantic duplicates** (different syntax, same behavior): write
     parallel tests against both sites with the same inputs; show they
     produce equivalent outputs.
   - **Reimplements existing utility**: grep for the utility's
     definition; show the new code does the same thing the utility
     already does.
3. Reject as false positive if the "duplication" is just standard
   boilerplate (DI constructors, getters/setters, framework lifecycle
   hooks, generated code).

**Verdict:**
- VERIFIED: duplication is real and the semantic equivalence holds.
- DISPROVEN: the two blocks look similar but have meaningful differences
  (different inputs, different return shapes, different side effects).
  Document the divergence.
- INCONCLUSIVE: cannot establish semantic equivalence (e.g., one side
  has runtime effects that can't be reproduced cleanly).

**Severity guidance:**
- `medium` for clear copy-paste between two changed files
- `medium` for new code reimplementing an existing utility
- `high` for near-duplicates with subtle differences — these often hide
  bugs where a fix was applied to one copy and missed in another
- `low` for cosmetic/idiomatic repetition where extraction would harm
  clarity (enum value mappings, switch arms)
- Avoid `critical` — duplicate code is rarely critical in itself, though
  it can enable other bugs (which would be reported under those classes)

**Failure modes:** False positives on standard boilerplate. The verifier
must reject candidates where the "duplication" is mechanical (framework
lifecycle methods, dependency injection wiring, ORM boilerplate) rather
than business logic.

---

## off_by_one

**Goal:** Show index/range arithmetic produces wrong bounds.

**Method:**
1. Write a test with boundary inputs (length 0, 1, N-1, N).
2. Assert the indices/ranges are correct.

**Verdict:**
- VERIFIED: test fails at a boundary.
- DISPROVEN: all boundaries pass.

---

## integer_overflow

**Goal:** Show arithmetic overflows or loses precision.

**Method:**
1. Write a test with large inputs (MAX_INT, MAX_INT - 1, MAX_INT * 2).
2. Check the result for wraparound, NaN, Infinity, or precision loss.

**Verdict:**
- VERIFIED: overflow observed.
- DISPROVEN: handled with bigint / bounded correctly.

---

## money_calculation

**Goal:** Show float math produces wrong currency amounts.

**Method:**
1. Write a test with inputs known to expose float issues:
   `0.1 + 0.2`, repeated additions, large × small.
2. Assert the result equals the expected exact value (in cents or with proper
   rounding).

**Verdict:**
- VERIFIED: result differs by ≥ 1 cent from expected.
- DISPROVEN: rounding is correct.

---

## timezone

**Goal:** Show datetime is mishandled w.r.t. timezones.

**Method:**
1. Write a test that constructs a datetime in one timezone, passes it through
   the path, and observes the output.
2. Try inputs around DST transitions, leap seconds, and the date line.

**Verdict:**
- VERIFIED: output is in the wrong timezone or skips/duplicates an hour.
- DISPROVEN: handled correctly.

---

## regex_dos

**Goal:** Show a regex has catastrophic backtracking.

**Method:**
1. Identify the regex.
2. Construct a pathological input:
   - For `(a+)+`: input `aaaaaaaaaaaaaaaaaaaaaa!`
   - Use known patterns; the verifier can search for "regex DoS payloads" via
     WebFetch if needed.
3. Run the regex with `timeout 5s`.

**Verdict:**
- VERIFIED: regex hangs or takes seconds for short input.
- DISPROVEN: regex completes quickly.

---

## supply_chain

**Goal:** Show a dependency has a known vulnerability or is unpinned.

**Method:**
1. Read the lockfile (`package-lock.json`, `Cargo.lock`, `go.sum`, `Gemfile.lock`,
   `requirements.txt`, `poetry.lock`, `pnpm-lock.yaml`).
2. Compare against advisories:
   - `npm audit --json`
   - `pip-audit --json`
   - `cargo audit --json`
   - `bundle audit`
   - `govulncheck ./...`
3. For unpinned: check `package.json` ranges, `Gemfile` looseness.

**Verdict:**
- VERIFIED: tool reports a CVE applicable to a pinned version.
- DISPROVEN: no advisories apply.
- INCONCLUSIVE: tool unavailable.

---

## other / unknown class

If the analyzer used `class: "other"`:
1. Read the `repro_strategy` field on the candidate.
2. Adapt one of the above recipes if it fits.
3. If genuinely novel, write the most direct reproduction you can think of:
   construct inputs, run, observe.
4. If you cannot construct a deterministic reproduction, return INCONCLUSIVE
   with a precise explanation of what's missing.

---

## Universal rules

- **Never** return VERIFIED without an actual run and captured output.
- **Always** include the exact command, exit code, and at least the first 50
  lines of output as evidence.
- **Prefer** the project's existing test framework over ad-hoc scripts.
- **Use timeouts** (`timeout 60s` or framework equivalents) on every run.
- **Don't** fix the bug while verifying. The proposed fix goes in the
  `proposed_fix` field; the repro should fail on the unfixed code.
- **Worktree is yours.** Install deps, scaffold tests, create files. It
  auto-cleans.
