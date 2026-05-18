---
name: critical-review-dispatch
description: Pre-review gates + 3-reviewer parallel panel dispatch for Critical/High risk tasks. Owns Design Agent, Verification Agent, Security Linter, Static Analysis Gate, and reviewer panel mechanics.
---

# Critical / High Review Dispatch

The Tasker loads this skill for Critical and High risk tasks. It owns:

1. Design Agent dispatch and design selection (Phase 2.3)
2. Pre-review gates (Phase 3.0): Verification Agent, Security Linter, Static Analysis Gate
3. 3-reviewer parallel panel dispatch (Phase 3.3) including plugin invocation and mid-flight failure handling

---

## Phase 2.3: Design Agent

Mandatory for Critical/High; optional for Medium; skip for Low. If `$SKIP_DESIGN` is set, skip entirely (Phase 1 analysis still runs).

### Dispatch

Use the Task tool with `subagent_type: general-purpose`:

```
Read the file `.claude/workflow/roles/design-agent.md` for your complete role instructions.
Risk level: [Critical/High]
Task Assignment: [paste full Task Assignment]
Reference implementations to study: [paste file paths from Phase 1]
```

### Select a Design

The agent returns 2-3 competing designs with trade-offs. **Tasker selects one** based on fit with codebase patterns, simplicity and correctness, sub-agent findings (financial invariants, schema — Critical only), and judgment as technical lead. If you cannot choose, escalate with the options laid out — do NOT flip a coin. If all designs carry unacceptable risk, escalate immediately.

### Pass Design to Coder

Append as `### Approved Design Spec` in the Task Assignment with this priority clause:

> The approved design takes precedence over the original spec. On a **meaningful conflict** (different approach, data model, or behavior) the Coder does NOT pick a side — they stop and present the divergence: (1) what the spec says, (2) what the design says, (3) where they conflict, (4) recommendation. Tasker escalates to human. For **minor deviations** (naming, parameter order, implementation details), follow the design and document under `⚠️ Interface Deviations`.

---

## Phase 3.0.1: Verification Agent (Critical only)

Dispatch **before** any reviewer. Produces independent ground truth — the agent does NOT see the Coder's Completion Report.

```
Read the file `.claude/workflow/roles/verification-agent.md` for your complete role instructions.
Branch: [branch-name]
Files changed (from Completion Report): [list]
Risk tier: Critical
Project commands (from CLAUDE.md): Build / Test / Lint / Complexity / Static analysis (gosec + staticcheck + semgrep) / Mutation if financial / Benchmark if benched.
Run from a clean worktree. Stop on first failure. Report raw output.
```

**FAIL** → return to Coder with failing step + raw output. Do not dispatch any reviewer. Iteration counter advances.
**PASS** → proceed to 3.0.2.

---

## Phase 3.0.2: Security Linter (Critical only)

```
Read the file `.claude/workflow/roles/security-linter.md` for your complete role instructions.

Risk context: [what this code touches — SQL, auth, money, etc.]

Files to audit:
[list files from Completion Report]
```

Verdicts:
- **PASS** → proceed to Phase 3.3 reviewer panel
- **FAIL** (Critical/High findings) → remediation path below
- **FLAG** (Medium findings, no Critical/High) → present to human for accept/reject

### Security Linter FAIL — remediation path

Do NOT proceed to reviewer panel. Send a targeted security fix request to Coder:

```markdown
## Security Fix Required: [Task ID]
**Security Linter Verdict:** FAIL
**Linter Cycle:** N/2

### Vulnerabilities Found (Must Fix)
- [ ] [File:Line] [Vulnerability] — surface: [SQL Injection / PII Exposure / Integer Overflow / Auth Bypass]
  - **Attack vector:** [how it could be exploited]
  - **Suggested fix:** [from linter output]

### Instructions
1. Fix ONLY the vulnerabilities listed above
2. Each fix must NOT introduce new attack surface
3. Re-run tests, confirm no regressions
4. Submit updated Completion Report listing ONLY the files you changed
```

Re-run the Security Linter on changed files after the fix.

**Cap at 2 linter cycles.** If still FAIL after 2 attempts, escalate to human — persistent vulnerability needs a design-level discussion, not another code fix.

Linter cycles are separate from review iteration cycles. A task can use 2 linter cycles AND have its full `MAX_ITERATIONS` (default 2) review iterations available — they address different concerns (exploitability vs. correctness).

If `$SKIP_SECURITY_LINTER` is set, skip this phase. Verification Agent still runs for Critical.

---

## Phase 3.0.3: Static Analysis Gate (High only)

For High risk (Critical already covers this via Verification Agent), run static analyzers as a Tasker-side gate:

```bash
gosec ./...
go list ./... | grep -vE '/pb$|/sqlc$|/mock$|/migration$' | xargs staticcheck
semgrep --config [project semgrep rules path] --error
```

Zero findings allowed. Suppressions in changed files are noted for the reviewer to validate; a suppression without rationale comment is a finding.

**Any analyzer fires** → return to Coder. Iteration counter advances.

> **Why High runs this Tasker-side** rather than trusting Coder's Phase 4.5 output: Tasker's job is to verify, not trust. Re-running the cheap deterministic checks catches pasted-but-not-actually-run output.

---

## Phase 3.3: Three-Reviewer Panel (Parallel)

All three reviewers receive the same Review Request, have **NO knowledge of the others' findings**, and produce independent 8-dimension reviews. Dispatch in parallel (same message). The prompt body is identical; only the transport differs:

- **Reviewer A** — Task tool with `subagent_type: general-purpose`.
- **Reviewer B** — Codex Claude Code plugin.
- **Reviewer C** — cc-gemini-plugin (Gemini).

Prompt body (use for all three):

```
Read the file .claude/workflow/roles/reviewer.md for your complete role instructions.
You did NOT write this code. Your job is to find defects, teach principles, and raise the bar.

[paste Review Request — spec, file list, actual test output]

Produce the full review output format from reviewer.md.
```

### Plugin Unavailability — Fallback

If either plugin is unavailable, dispatch an additional Claude subagent as replacement reviewer. Two-reviewer consensus is the minimum for High; three is required for Critical.

If `$REVIEWER_COUNT` is set, honor that count (overrides tier default).

### Mid-Flight Reviewer Failure

On plugin crash, API outage, malformed output, or excessive time:

1. **Re-dispatch once** with the same Review Request — transient failures often resolve on retry.
2. **If re-dispatch also fails**, fall back to N−1 / N consensus with the gap flagged in the merged review: `⚠️ Reviewer C (Gemini) unavailable after retry. Consensus is 2/3.`
3. **Tier handling:** Critical — 2/3 NOT acceptable by default; stop and report to human, who decides accept / wait / escalate. High — 2/3 is the floor; proceed but flag. Medium — single-reviewer is fine; if it failed twice, retry later or escalate.
4. **Never silently proceed** with fewer reviewers than configured. Every absence is visible in the merged report.

---

## Component-specific dimension floors

The floor table is canonical in `tasker.md` Phase 4.2. Why each floor was set where it was:

- **Wallet / settlement / placement writes (Perf 5, Idem 5, Resil 5).** Money-movement paths at high TPS with real concurrent-duplicate risk. 5/5 on performance, idempotency, and resilience reflects what correctness means at this scale, not aspirational engineering. A 4/5 here means "good enough most of the time" — which is the same as "duplicate-pays under load."
- **Bet settlement also requires Obs 5.** Dispute resolution and regulatory inquiries need the full bet lifecycle reconstructed (placed → odds locked → event resolved → outcome determined → payout calculated → wallet credited) with timing at each state transition. 4/5 observability is "logged the result" — not enough. 5/5 is "I can answer the player and the regulator without paging an SRE."
- **Jackpot award paths (Perf 4, Idem 5, Resil 5).** Low TPS but catastrophic per-event risk — every duplicate jackpot pays out the full prize, every dropped jackpot is a regulatory incident. Idem and Resil 5/5 are non-negotiable; Perf 4/5 (bounded queries, documented indexes, no obvious N+1) is sufficient given the rate.
- **Responsible gambling enforcement (Obs 5).** The compliance pass/fail dimension covers "did you log it?" — observability 5/5 covers "can you reconstruct the decision?" Regulators require the full decision path: what checks ran, what data was consulted, what the exclusion status returned, why a player was allowed through (or stopped). Anything below 5/5 here makes audit responses qualitative when they need to be reproducible.

If a reviewer scores a floor dimension at less-than-floor, the iteration request must name the specific gap (which call site, which log statement, which test) — not a generic "raise the score." The Coder targets the gap; the re-review verifies the gap closed.

---

## Review Request Format

The Reviewer receives this — never include the Coder's self-assessment:

```markdown
## Review Request
**Task ID:** [from assignment]
**Risk:** Critical/High/Medium/Low

### Original Spec
[Copy Task Assignment — Objective through Definition of Done]

### Implementation
[List of files to review]

### Test Output
[Paste actual test output]

### Files for Review
[Code files attached / paths listed]
```

> Pre-review gates already cleared by the time you dispatch the panel — Critical has Verification Agent + Security Linter PASS; High has Static Analysis Gate PASS. Reviewers audit code that's already verified to compile, test green, lint clean, and pass deterministic security/quality scans.
