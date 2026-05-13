# Severity rubric

Verifiers return `severity_final` in their JSON. The orchestrator normalizes
to these four canonical levels before writing the report.

## critical

A real, exploitable, or production-fatal defect. One of:

- **Data loss**: irreversible state corruption that affects real users
- **RCE / arbitrary code execution**: attacker can run code on the server
- **Auth bypass on a privileged path**: unauthenticated access to admin or
  user-data endpoints
- **Money calculation off**: wrong amounts billed, refunded, or transferred
- **Production crash on common path**: every request crashes or every user
  hits the bug
- **Credential / secret exposure**: secrets appear in logs, responses, or
  error messages that reach users
- **Stored XSS / second-order injection**: attacker persists payload that
  fires on other users

Critical findings should never be ignored. The report leads with these.

## high

A real defect that produces wrong results or crashes for a meaningful subset
of users:

- **Wrong output on a common code path** (not all users, but many)
- **Crash on inputs a real client will send** (e.g., null, empty array,
  large string)
- **Reflected XSS / SQL injection** that requires user interaction
- **N+1 query** that makes a page noticeably slow at production scale
- **Resource leak** that exhausts the process within hours of normal traffic
- **Race condition** that produces wrong state and was reproducible in repro
- **Off-by-one** at a boundary users hit
- **SSRF / path traversal** with limited blast radius

## medium

A real defect with limited scope or rare triggers:

- **Edge-case crash** that requires unusual but valid inputs
- **Perf regression** that's measurable but not catastrophic
- **Resource leak** under stress that wouldn't trigger in normal use within
  the deploy cycle
- **Race condition** that's reproducible but rare in production
- **Type error** caught at compile time (won't reach runtime, but signals
  fragile code)
- **Error swallowing** that hides debuggability without breaking behavior
- **Dead code** in a non-vendored, owned module (worth removing; not urgent)
- **Timezone bug** affecting display only, not data

## low

A real defect with minimal user impact:

- **Logic flaw with graceful failure** (e.g., returns a default when it
  shouldn't, but no crash)
- **Cosmetic data leak** (e.g., timing info in error message — useful for
  attackers but not directly exploitable)
- **API misuse that works but is fragile** (e.g., relying on undocumented
  behavior)
- **Dead code** in a vendored or test-only path
- **Inconsistent error format** that affects API consumers' parsing

## Not reported (filtered out)

These never appear in the report regardless of class:

- Style preferences ("use const here")
- Naming opinions
- "Could be more idiomatic" suggestions
- "Consider adding tests" notes
- Documentation gaps
- Lint warnings the project doesn't enforce
- Any concern that wasn't VERIFIED

These categories are explicitly the *opposite* of what goobreview produces.
Style review is a different tool.

## Normalization rules

If a verifier returns:
- `critical`, `crit`, `severe`, `urgent` → `critical`
- `high`, `important`, `major` → `high`
- `medium`, `moderate`, `minor` → `medium`
- `low`, `trivial`, `nit` → `low`

If unclear or missing, default to `medium` and note this in the report.

## Severity vs. confidence

These are different axes. Severity is "how bad is this if real?". Confidence
is "how sure are we it's real?". Verification handles confidence. By the time
a finding enters the report, confidence is high (it has a runnable repro).
So the report only shows severity, not confidence.

Disproven and inconclusive candidates do appear in appendices but without
severity (they're not actionable findings).
