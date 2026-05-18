---
name: iteration-protocol
description: Targeted re-review after Coder fixes CRITICAL/HIGH findings. Owns the iteration cap, full domain test gate, remediation ordering, and the iteration-request format.
---

# Iteration Protocol

Load this skill on `ITERATE` verdict from the review panel. The skill owns the loop mechanics: how to format the fix request, the gate before re-review, the targeted re-review scope, and the iteration cap.

---

## The Loop

```
ITERATE verdict
  ↓
Send CRITICAL/HIGH-only fix request to Coder (format below)
  ↓
Coder fixes — minimal diff, no refactoring
  ↓
Coder runs full domain test suite — must be GREEN
  ↓
Coder returns updated Completion Report
  ↓
Tasker verifies full domain suite output independently
  ↓
Targeted re-review (changed files only)
  ↓
APPROVE → pr-raise.md   |   ITERATE → loop (max 2)   |   REJECT → escalate
```

---

## Iteration Cap

- **Max 2 cycles** per task (initial review + 1 fix round), or `$MAX_ITERATIONS` if set in the environment.
- If open CRITICAL or HIGH findings remain after the cap, write `Status: Blocked` with reason "iteration cap reached" and include the full review history in the summary's Escalation reason.
- The cap recommendation in the summary: pair session, spec clarification, or scope reduction.

> **Why 2 and not 3:** if the Coder cannot resolve a finding after one targeted fix iteration, more iterations rarely help — they typically introduce regression or scope creep. Two cycles also bounds dispatcher runtime more tightly for unattended runs. Override with `--max-iterations 3` if the human believes a specific class of finding genuinely needs the extra cycle.

---

## Fix Request Format

Send this to the Coder — never include MEDIUM/LOW findings in the actionable list:

```markdown
## Iteration Required: [Task ID] — Round N/2

**Tier minimum:** every quality dimension ≥ [4/5 for Critical/High, 3/5 for Medium] with no CRITICAL/HIGH findings open
**Current weakest dimension:** [name] at [score]/5 from [reviewer(s)]

### CRITICAL Findings (Must Fix)
- [ ] [File:Line] [Finding] — flagged by: [reviewers]

### HIGH Findings (Must Fix)
- [ ] [File:Line] [Finding] — flagged by: [reviewers]

### MEDIUM/LOW Findings (Logged — DO NOT Fix Now)
These are tracked for future work. Fixing them in this iteration risks regressions.
- [ ] [File:Line] [Finding] — [severity]

### Instructions
1. Fix ONLY the CRITICAL and HIGH findings above
2. Each fix must include a regression test proving existing behavior still works
3. Do NOT refactor surrounding code while fixing — minimal diff only
4. Run the full domain test suite and paste the output in your Completion Report:
   - Wallet:      `go test -race ./apps/finance-domain/wallet/...`
   - Engine/Game: `go test -race ./apps/game-domain/...`
   - Platform:    `go test -race ./apps/platform-domain/core/...`
5. Submit updated Completion Report listing ONLY the files you changed
```

This is the **only fix round** under the default `MAX_ITERATIONS=2` cap. If your fix doesn't resolve the findings cleanly, the task is Blocked next round — the Tasker writes an Escalated/Blocked summary rather than iterating again. Treat this as your one shot.

---

## Remediation Ordering

When multiple findings need fixing in one iteration:

1. Fix all CRITICAL findings first (small, targeted changes).
2. Fix HIGH findings next.
3. Commit and verify with a quick local re-run after each group.
4. **NEVER mix bug fixes with refactoring** in the same iteration.
5. Each fix has a minimal diff — do not "improve" surrounding code.

If a fix in step 1 makes a step-2 fix harder, stop and report back — do not improvise. The Tasker may decide to re-scope the iteration.

---

## Full Domain Test Suite Gate (before re-review)

A targeted fix to one file can regress a sibling file whose tests are not run by changed-files-only test commands. Catching that regression with the test suite is cheap; catching it post-merge is expensive.

**Before dispatching the targeted re-review:**

- The Coder's iteration Completion Report includes the full domain-scoped test command output (from step 4 of the Fix Request).
- The Tasker independently verifies that output — re-run from the worktree if the output looks suspicious.
- If any test fails: the iteration is rejected and returned to the Coder. Do NOT dispatch the targeted re-review on broken code.

This separates "are tests still passing across the domain" (Tasker's deterministic job) from "did the reviewer audit the universe" (which the workflow correctly avoids to prevent regression spirals).

---

## Targeted Re-Review

After the domain test gate is GREEN, send a **scoped** re-review — NOT a full audit:

```markdown
## Targeted Re-Review: [Task ID] — Round N/2

### Scope: Changed files only
[list of files changed by Coder in this iteration]

### Previous Findings Being Verified
[list of CRITICAL/HIGH findings with file:line references from the prior review]

### Domain Test Suite Status
✅ Full domain suite verified GREEN by Tasker before dispatch — you are reviewing code quality, not test sufficiency.

### Instructions
1. Verify each previous CRITICAL/HIGH finding is resolved
2. Check for regressions in changed files ONLY
3. Do NOT audit unchanged files for new issues
4. For each prior finding, report: RESOLVED / STILL OPEN / REGRESSED
5. New findings in changed files: categorize as CRITICAL/HIGH/MEDIUM/LOW
6. Only new CRITICAL/HIGH findings trigger another iteration
```

Continue until no open CRITICAL/HIGH findings remain, or the iteration cap is reached.

For Critical/High: dispatch all 3 reviewers (or the configured `$REVIEWER_COUNT`) in parallel for the targeted re-review, same as the initial review. The fallback / mid-flight retry rules in `critical-review-dispatch.md` apply.

For Medium: a single reviewer for the targeted re-review is sufficient — they're verifying their own prior findings.

---

## Verdict at Each Cycle

- All previous CRITICAL/HIGH findings RESOLVED + no new CRITICAL/HIGH in changed files → APPROVE
- Any prior finding STILL OPEN or REGRESSED → ITERATE (with the still-open findings listed in the next Fix Request)
- Any new CRITICAL/HIGH found in changed files → ITERATE (added to the next Fix Request)
- After Round 2 with open CRITICAL/HIGH → write `Status: Blocked`, reason "iteration cap reached", include full review history (or after `$MAX_ITERATIONS` if overridden)

---

## What NOT to Do

- Do not iterate on MEDIUM/LOW. They are logged in the summary's `Deferred findings` section. Fixing MEDIUM/LOW in a fix iteration risks regression because the Coder mixes scope.
- Do not run a full re-audit. The cost of re-reading unchanged files for new issues compounds across iterations and creates a regression spiral — every "new finding" in unchanged code restarts the process.
- Do not skip the full domain test suite gate. The cost is one test run; the cost of skipping it is finding regressions post-merge.

---

## Common Failure Modes

| Failure | Cause | Fix |
|---------|-------|-----|
| Re-review finds a new MEDIUM in changed files | Coder's fix touched surrounding code | Reject the iteration; require minimal diff |
| Domain tests fail after the fix | Fix regressed a sibling file | Iteration is rejected back to Coder; do not dispatch re-review on red |
| Reviewer audits unchanged files | Re-review prompt was unclear | Re-state the scope; the prompt template in this skill explicitly scopes the audit |
| Iteration count drifts above the cap silently | Tasker lost track of cycles | Track explicitly in the summary file's `Iterations` field; cap at `$MAX_ITERATIONS` (default 2) |
| Coder bundles a refactor into an iteration | Treated the iteration as a general code-quality pass | Reject; iterations fix bugs, not style |
