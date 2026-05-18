---
name: git-worktree-setup
description: One ticket = one worktree = one branch = one PR. Container vs host path conventions, branch naming, what belongs in the repo vs Jira.
---

# Git Worktree Setup

Load this skill before dispatching the Coder for the first time on any ticket. Every ticket gets its own worktree — no exceptions.

---

## The Rule

**One ticket = one worktree = one branch = one PR.**

- Branch from `main` only
- One ticket's work per branch — never combine tickets
- The PR for that branch targets `main`
- The PR is raised only after the task passes review (see `pr-raise.md`)

Work is never done directly on `main` or on a shared branch.

---

## Branch Naming

| Task type | Branch name |
|-----------|-------------|
| Bug fix | `fix/SMG-XXXX-short-description` |
| Feature | `feat/SMG-XXXX-short-description` |
| Refactor | `refactor/SMG-XXXX-short-description` |
| Docs | `docs/SMG-XXXX-short-description` |
| Chore / infra | `chore/SMG-XXXX-short-description` |

Short description is kebab-case, ≤ 5 words, summarizing the change.

---

## Where to Put the Worktree

The Tasker (and dispatcher, if in dispatcher mode) chooses the path based on the runtime environment.

### Container environment

Detected when the repo root is `/workspace`. The parent directory of `/workspace` is **not writable** — `git worktree add ../...` fails with permission errors.

**Use `/worktrees/<ticket-id>` instead.** That path is a writable tmpfs mount provided by the container.

```bash
git worktree add /worktrees/SMG-2384 -b feat/SMG-2384-bay-session-state
```

The worktree is ephemeral (in-memory) — it exists only for the container's lifetime. That's fine: all work is committed and pushed before teardown. The main `/workspace` checkout remains on its original branch and is not modified by sub-agent work.

### Host environment

Detected when the repo root is not `/workspace`. Use the sibling-directory convention:

```bash
git worktree add ../worktree-SMG-2384 -b feat/SMG-2384-bay-session-state
```

The Coder runs in that directory; the main checkout stays on its branch.

### Dispatcher convention

When running under the dispatcher, the dispatcher itself creates the worktree at the path it chose (`/worktrees/<task-key>` in container mode, `../worktree-<task-key>` on host) and passes the path to the Tasker via env or in the Task Assignment. The Tasker does not re-create the worktree — it uses the one the dispatcher provided.

---

## Dispatching the Coder Into a Worktree

In the Task Assignment, name the working directory explicitly:

```markdown
**Working directory:** /worktrees/SMG-2384
**Branch:** feat/SMG-2384-bay-session-state
```

The Coder runs `[project test command]`, `[project lint command]`, and all git operations inside that directory. The main checkout is not touched.

---

## What Belongs Where

| Belongs in the repo | Belongs in Jira / local only |
|---------------------|-------------------------------|
| Code, tests, migrations | Ticket descriptions, investigation notes |
| README, architecture docs | Task YAML files used for planning |
| `.claude/workflow/roles/` and `.claude/workflow/skills/` | Spike / scratch work |
| Migrations + their down counterparts | Per-task summary files (these go to `docs/runs/<run-id>/` under dispatcher, or stdout standalone) |

**Never commit Jira ticket content, task management YAMLs, or investigation notes to the repository.** A Completion Report that adds `*.yaml` for task tracking, `*-notes.md` for investigation, or `*-handoff.md` is rejected — those belong in Jira or the dispatcher's run directory.

---

## Worktree Lifecycle

| Stage | Action |
|-------|--------|
| Pre-dispatch | Tasker (or dispatcher) creates worktree from `main`; branch named per the convention above |
| During work | Coder commits to the branch; no rebase or merge from `main` mid-task |
| Post-approve | PR raised from the branch (see `pr-raise.md`); branch lives until PR merges |
| Post-merge | Branch can be deleted; worktree removed with `git worktree remove <path>` |
| Failure / Blocked | Worktree preserved for inspection — do NOT auto-remove on Blocked status |

The dispatcher (if used) records the worktree path in its run log so subsequent commands (`dispatcher resume`, `dispatcher report`) can locate it.

---

## Common Failure Modes

| Failure | Cause | Fix |
|---------|-------|-----|
| `git worktree add ../worktree-X` permission denied | Running in a container where `/workspace`'s parent isn't writable | Use `/worktrees/<ticket-id>` instead |
| Branch name collision | Reused a name from a previous ticket | Re-create with the full `SMG-XXXX-description` form |
| Coder commits to `main` directly | Skipped worktree creation | Reset their work into a new worktree; do not amend `main` |
| PR bundles two tickets | Multiple tickets worked in the same worktree | Reject the PR; split the work into separate branches |
| Investigation notes committed to repo | Coder treated the worktree as a notebook | Reject the Completion Report; instruct removal before PR |
