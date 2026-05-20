---
name: prd-to-task-yaml
description: Convert a PRD / spec / requirements document into a dispatcher-ready tasks YAML. Identifies foundational vs leaf work, sets blockedBy chains, picks sizes + models per task, and produces a YAML the dispatcher can run. Load this when the user asks to turn a PRD into a task list.
---

# PRD → Task YAML

Load this skill when the user hands you a PRD / spec / requirements doc and asks for a dispatcher-ready task YAML. Output: a single YAML file conforming to the schema described in `forecast-fields.md` (load that skill alongside if the project uses the `forecast` Jira bridge).

This skill never executes the work — it produces a plan. Pair with `task-yaml-review.md` to validate the output before the human ships it to the dispatcher.

---

## Inputs

- The PRD / spec document path (read it in full first; do not skim).
- The project context: existing task-key namespace (e.g. `BSA-`, `SMG-`, `BSA-E2E-`), existing migration / module structure, the codebase areas the work touches.
- Optional: a target task count or a per-task size cap. Default to "each task ≈ one PR's worth of focused work" — typically S–M, occasionally L for inseparable units.

## Outputs

- A `<project>-<feature>-tasks.yaml` written to the repo root (or wherever the user names).
- A short narrative summary printed to the user: foundation chain, leaf wave shape, total tasks, optimisation regime applied.
- **Do NOT dispatch the tasks** — produce the YAML and stop.

---

## Step 1: Read + classify

Open the PRD. Pull out every discrete unit of work — anything that could be one PR. For each unit, classify:

### Foundation vs leaf

- **Foundation** = used by ≥ 2 downstream units. Schema migrations, shared types/libraries, base infrastructure (test harness scaffolding, shared assertion modules, DB seed routines, common config), auth surface.
- **Leaf** = no downstream dependents. Independent feature implementations, individual scenarios, isolated docs, observability dashboards.

Foundation forms a **chain** (serialized via `blockedBy`). Leaves form a **wave** (all `blockedBy` the last foundation task; they can run in parallel).

### Risk classification (drives model + parallelism floor)

Use the same tiers the dispatcher applies for review depth:

| Risk | Applies to | Model | Parallelism |
|---|---|---|---|
| **Critical** | balance mutation, settlement, payouts, gambling outcome, withdrawals, anything in `--financial-paths` glob | Opus | within-wave serial (1-at-a-time) |
| **High** | auth surface, state machines, audit trails, ledger, public API contracts | Opus | at most 2 in parallel |
| **Medium** | repositories, validation, queries, recovery workers, read-only services | Opus for foundation; Sonnet for leaves | 4+ in parallel for leaves |
| **Low** | config, docs, test helpers, observability, migrations with no FK | Sonnet | unlimited parallel |

**Quality floor** (NEVER violate): Critical + High → Opus. Foundation tasks → Opus regardless of risk (a mistake at the foundation propagates to every downstream task; cost difference is rounding error vs the cascade-failure cost).

### Size (`size:XS | S | M | L | XL`)

Estimate by LOC + review depth:

| Size | LOC band | Typical scope | Review |
|---|---|---|---|
| XS | < 50 | One-file fix, single seed migration, doc tweak | self-review |
| S | 50–250 | Single function + tests, single schema migration with tests, one RPC handler | single reviewer |
| M | 250–600 | New service file or two, integration handler + harness changes, multi-table migration with cross-table invariants | 2 reviewers |
| L | 600–1200 | Whole subsystem (worker + manifests + tests + observability), public RPC + handler + tests, multi-file refactor | 2–3 reviewers + design |
| XL | > 1200 | Multi-component (e.g. service decommission across smg-core + bay-session + cloud-frs); **avoid — split into multiple L** | usually NOT shippable; split first |

If a unit looks XL, split it. Multiple Ls run in parallel; one XL serializes the whole epic.

## Step 2: Build the dependency graph

For each task, fill `blockedBy:`:

- **Foundation chain**: F1 → F2 → F3 → ... each `blockedBy: [F<prev>]`.
- **Leaf wave**: every leaf `blockedBy: [F<last>]` (or specific subset if some leaves don't need the full foundation).
- **Cross-leaf deps**: only when leaf B genuinely needs leaf A's code. Most leaves should be independent. If you find yourself chaining leaves, ask whether you've labeled foundation wrong — that leaf might actually be foundation.

**Sanity check**: `for each task, leaves not in foundation_chain MUST NOT blockedBy each other.` If two leaves chain to each other, they're not independent and one of them is foundation.

## Step 3: Per-task fields

For each task, write:

```yaml
- key: <project>-<feature>-<group>-<seq>           # e.g. BSA-E2E-0-1
  summary: "<imperative present, ≤ 80 chars>"      # what gets DONE
  description: |
    <2-5 paragraph scope: what to build, the contract, the acceptance
    criteria. Reference spec sections by §. Spell out files the task
    is allowed to touch — Taskers default to over-reach without this.>

    Acceptance:
      - <concrete observable; tests pass / file exists / table present>
      - ...

    Out of scope:
      - <explicit non-goals; Taskers will scope-creep otherwise>
  type: Task
  estimate: <Nh>                                   # hours; lossy but useful
  labels: [size:<S|M|L>, area:<dir>, area:<theme>]
  blockedBy: [<key>, ...]
  model: <claude-opus-4-7 | claude-sonnet-4-6>     # see Step 1 risk table
  jira_key: ""                                     # forecast bridge fills this in
```

### Description writing rules

The description is the Tasker's full context. Write it like you would brief a competent colleague who hasn't read the PRD:

- **Lead with intent**: 1–2 sentences on what user-facing capability this delivers.
- **Cite the spec section(s)** the task implements.
- **List files allowed to touch** (or "no files outside `apps/X/`"). Without this, Taskers refactor adjacent code.
- **Spell out the public contract** if there is one (RPC shape, schema columns, error codes). Vague descriptions → wasted Tasker iterations.
- **Acceptance criteria as a bulleted list**, each item testable.
- **Out-of-scope bullets** for anything a reasonable Tasker might assume is included but isn't. This is the most under-used field; use it liberally.

### When in doubt on model

| Heuristic | Pick |
|---|---|
| Task touches `--financial-paths` (wallet, settlement, payouts, recovery) | **Opus** (forced by quality floor) |
| Task is foundation (≥ 2 downstream) | **Opus** |
| Task is HIGH risk per the table above | **Opus** |
| Task is a Medium leaf in non-financial code | depends on optimisation target — see Step 4 |
| Task is Low risk (docs, isolated test, config) | **Sonnet** |

## Step 4: Optimisation regime

Ask the user once: `optimise for time, money, or balanced (default)?`

### Quality floor (applied first, regardless of regime)

- Critical + High → Opus
- Foundation → Opus
- Financial-paths glob → Opus
- 3-model-panel triggers → Opus (per `critical-review-dispatch`)

These are non-negotiable. The optimisation regime only chooses among tasks NOT pinned by the floor.

### Time optimisation

For tasks not pinned by the floor:

- Prefer **Opus** even for Medium leaves — Opus's higher per-task quality reduces iterations, which dominates wall-clock for tasks that need review-fix cycles.
- Increase `--max-parallel` to the maximum the host can sustain (≥ 4) for leaf waves.
- Keep foundation chain serial; ordering wastes wall-clock if a downstream task forks from stale base.

### Money optimisation

For tasks not pinned by the floor:

- Prefer **Sonnet** for Medium leaves that don't touch financial paths.
- Prefer **Sonnet** for all Low-risk tasks.
- Use `--max-parallel 1` if cache costs are a concern — concurrent workers can't share the prompt cache, so a Sonnet at parallel-4 can cost more than an Opus at parallel-1 for cache-hot work.
- For known-flaky tasks (rare model output, deep tool-use chains), keep Opus to avoid retry cost.

### Balanced (default)

For tasks not pinned by the floor:

- Foundation + Medium-foundation-adjacent → Opus
- Medium leaves → Sonnet
- Low → Sonnet

This is the recommended default. Time-optimised is for "we have a deadline and the run will be reviewed end-to-end anyway." Money-optimised is for "I want to compare strategies and see the data."

## Step 5: Output

Write the YAML to disk. Include a comment header:

```yaml
# project-feature-tasks.yaml
#
# Generated by prd-to-task-yaml skill from <PRD path>.
# Optimisation regime: <time | money | balanced>.
# Foundation chain: <F1, F2, F3>.
# Leaf wave size: <N>.
# Recommended dispatch:
#   dispatcher run <file> --mode unattended --max-parallel <N> \
#     --base-branch <branch> --auto-integrate \
#     --claude-extra-args "--permission-mode bypassPermissions --allow-dangerously-skip-permissions"

project: <project>                                     # from forecast-fields.md schema if applicable
epic: <epic-key>
base_branch: <branch>
forecast_config: .forecast/config.yaml                 # optional, omit if no forecast

tasks:
  - key: <F1-key>
    ...
```

After writing, print to the user:

- Total task count + breakdown by size.
- Foundation chain (numbered list with sizes + models).
- Leaf wave summary (count + models).
- The recommended `dispatcher run` invocation.
- Any open questions the PRD left ambiguous that you flagged inside task descriptions for the Tasker to escalate.

## Step 6: Hand off

End the skill with:

```
Next step: run `task-yaml-review.md` against <output-path> to validate
the graph + optimisation choices before dispatching.
```

Do NOT invoke the dispatcher. Do NOT create Jira tickets — that's a separate `forecast-create` step the user runs after they're happy with the YAML.
