---
name: plan-based-execution
description: Plan-based vs free-form Coder dispatch. When a docs/plans/*.md exists, dispatch via superpowers:executing-plans with batch checkpoints; otherwise monolithic free-form.
---

# Plan-Based vs Free-Form Execution

Load this skill before dispatching the Coder. The choice between plan-based and free-form is binary:

- **`docs/plans/<plan>.md` exists for this task → plan-based**
- **No plan file → free-form**

---

## Plan-Based Execution (preferred when a plan exists)

### Dispatch

Instruct the Coder to use `superpowers:executing-plans`. Reference the plan file explicitly:

```markdown
## Task Assignment — Plan-Based

**Plan File:** `docs/plans/YYYY-MM-DD-feature-name.md`

**REQUIRED SUB-SKILL:** Use `superpowers:executing-plans` to execute this plan.

Execute the plan in batches of 3 tasks. After each batch, report back to me (Tasker) with what was done and any blocking issues. Wait for my approval before proceeding to the next batch.

[Include the full Context / Risk Level / Definition of Done from the Tasker Phase 2 template]
```

### Batch Checkpoint Protocol

After each batch the Coder reports back. You (Tasker) must:

1. **Verify** the batch outputs: files exist, tests pass, no regressions in changed files.
2. **Approve** (continue to next batch) OR **block** (stop and fix before proceeding).
3. **Do NOT dispatch full review for every batch** — full review happens once, after the final batch.

### Triggering a Batch Block

Block immediately if any of:

- Build breaks
- Tests fail (changed packages)
- Files missing that the plan said to create
- Obvious security issue visible without deep review (e.g., raw SQL string-concat, hardcoded credential)
- Coder skipped a plan step without flagging

If blocked, send a focused fix request to the Coder — minimal diff, just the failing step — and re-check that batch before approving.

### When All Batches Are Done

Treat the final report as a standard Completion Report and proceed to Phase 3 (Pre-Review Gates) onward in the Tasker workflow. The plan-based path does not skip pre-review gates or the review panel — it only delays the full review until the work is consolidated.

---

## Free-Form Execution (when no plan file exists)

Dispatch a single monolithic Task Assignment using the Phase 2 template from the Tasker role. The Coder runs the whole thing end-to-end and returns one Completion Report. Proceed to Phase 3 when they're done.

---

## Why Plan-Based at All

For multi-file or multi-component work where the plan was written ahead of time:

- The plan already encodes the breakdown — re-deriving it in the Task Assignment duplicates effort.
- Batch checkpoints catch drift early — a Coder veering off the plan in batch 1 is much cheaper to redirect than one who delivered all 9 tasks before review.
- The Coder context stays focused — they're executing a known plan, not interpreting a multi-paragraph spec.

For single-file or focused work, the overhead of plan-based execution outweighs the benefit — use free-form.

---

## Common Failure Modes

| Failure | Cause | Fix |
|---------|-------|-----|
| Coder runs full review on every batch | Misunderstood batch checkpoint protocol | Re-state: full review happens once at the end; batch checkpoints are Tasker-side verification only |
| Coder bundles all batches and reports once at the end | Skipped the report-back-after-each-batch step | Block; require the Coder to redo with explicit checkpoint reports |
| Batch 1 passes, batch 2 silently regresses batch 1 | Coder didn't re-run tests across changed packages | Block; require domain-scoped test re-run before approving the batch |
| Plan file conflicts with the Task Assignment | The plan was written before the spec was final | Stop, escalate to human — do not have the Coder pick a side |
| No batch reports during a long Coder run | Coder went heads-down for the full work | Wait for completion, then verify all artifacts; in future Task Assignments, repeat the batch-checkpoint requirement explicitly |

---

## Integration with Dispatcher Mode

The dispatcher does not change the plan-vs-free-form choice — it's purely based on plan file presence in the repo. The dispatcher's `MAX_ITERATIONS` env var applies to the final review cycles, not to batch counts. Batch checkpoints are not iterations and don't count against `MAX_ITERATIONS`.

In supervised dispatcher mode, batch blocks surface to the human if they can't be resolved by the Coder within one cycle. In unattended mode, a batch block that the Coder cannot resolve writes `Status: Blocked` with reason "batch N blocked: <reason>" and exits.
