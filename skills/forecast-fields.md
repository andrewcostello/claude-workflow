---
name: forecast-fields
description: YAML field schema that lets the dispatcher and the `forecast` Jira tool coexist on the same task list. Load this when writing or editing a tasks YAML that will round-trip through forecast for Jira ticket creation and status sync.
---

# Forecast-Field Schema

Load this skill when the Tasker is asked to write a new tasks YAML, or to add rows to an existing one, in a project that uses the `forecast` Jira tool. The schema below makes each row valid dispatcher input AND mappable to `forecast jira create` / `forecast jira transition` so the bridge can keep Jira in sync.

If the project doesn't use `forecast`, ignore this skill — the dispatcher itself only needs `key`, `summary`, `description`, `type`, `labels` (with `size:*`), and optional `blockedBy`.

---

## The schema

```yaml
project: SMG                                # informational; matches Jira project key
epic: BSA                                   # default epic for tickets in this file
forecast_config: .forecast/config.yaml      # which forecast config governs

# Optional bridge tuning (defaults shown):
forecast:
  placeholder_prefix: "TBD-"                # keys with this prefix get `forecast jira create`
  status_mapping:                           # YAML status -> Jira transition
    Done: Done
    Blocked: "Is Blocked"
    Escalated: "Is Blocked"
  # Override values may also be objects: `{to: "Resolved", resolution: "Won't Do"}`

tasks:
  - key: BSA-E2E-0-1                        # local identifier (dispatcher-side). Free-form,
                                             # unique within the file. Can be semantic
                                             # (BSA-E2E-0-1) or generic (TBD-1) — either works.
    jira_key:                               # Jira issue key. Left empty until the bridge runs
                                             # `forecast jira create` and writes the assigned
                                             # SMG-NNNN here. Set explicitly if you have an
                                             # existing Jira ticket you don't want the bridge
                                             # to re-create.
    summary: "Short imperative summary"     # REQUIRED — used by both
    description: |                          # REQUIRED — used by both
      Multi-line description.

      Acceptance:
        - point one
        - point two
    type: Task                              # REQUIRED — Bug / Story / Task / Sub-task / Epic
    labels:                                 # REQUIRED — must include exactly one `size:*`
      - size:M
      - type:component
      - area:schema

    # Optional forecast-mappable fields:
    priority: Medium                        # Highest / High / Medium / Low / Lowest
    epic: SMG-100                           # overrides top-level epic
    parent: SMG-100                         # for Sub-task type
    story_points: 5
    due_date: "2026-06-30"                  # YYYY-MM-DD, quoted
    assignee: andrew@evenplay.com
    fix_versions: [v1.4.2]
    components: [Backend, API]

    # Dispatcher-internal fields:
    blockedBy: [SMG-1230]                   # used by runnable-set computation
    estimate: 6h                            # human-readable, spec-readability only
```

---

## Field mapping

| YAML field | Required | `forecast jira create` flag | Jira field |
|------------|----------|-----------------------------|------------|
| `key` | yes | (not read or written by the bridge) | (not a Jira field — local identifier only) |
| `jira_key` | no (bridge writes it) | (output — Jira assigns) | Issue key |
| `summary` | yes | `--summary` | Summary |
| `description` | yes | `--description` | Description |
| `type` | yes | `--type` (default Task) | Issue Type |
| `labels` | yes (must include `size:*`) | `--labels` (comma-joined) | Labels |
| `priority` | no | `--priority` (default Medium) | Priority |
| `epic` | no | `--epic` | Epic Link |
| `parent` | no | `--parent` | Parent (Sub-task → Story) |
| `story_points` | no | `--story-points` | Story Points |
| `due_date` | no | `--due-date` | Due Date |
| `assignee` | no | `--assignee` | Assignee |
| `fix_versions` | no | `--fix-versions` (comma-joined) | Fix Versions |
| `components` | no | `--components` (comma-joined) | Components |
| `blockedBy` | no | (not a forecast flag) | "is blocked by" Link |
| `estimate` | no | (not a forecast flag) | (not Jira — readability only) |
| `status` + lifecycle fields | dispatcher-written | (not a forecast flag) | bridge calls `forecast jira transition` |

---

## Key vs jira_key (important)

The dispatcher and the bridge use two different fields:

- **`key`** — the dispatcher's local identifier. Used for `blockedBy` references, runnable-set computation, summary file paths, and `--only` filters. **Free-form** (must just be unique within the file). The bridge never reads or writes it.
- **`jira_key`** — the Jira issue key (e.g., `SMG-2890`). Empty until the bridge runs `forecast jira create`. Set explicitly if you already have a Jira ticket for that row.

This separation means semantic local keys like `BSA-E2E-0-1` survive the bridge unchanged. The bridge appends `jira_key: SMG-2890` to that row; `BSA-E2E-0-1` stays as the dispatcher identifier so `blockedBy: [BSA-E2E-0-1]` references downstream still resolve.

## How the bridge uses these fields

**`dispatcher forecast-create <yaml>`** — for each task row with no `jira_key`, runs `forecast jira create` with the mapped flags. On success, captures `Created: KEY-NN` from stdout and writes that key to `row["jira_key"]`. The `key` field is left untouched.

Idempotent. Re-running after a partial failure picks up only the rows that still lack `jira_key`.

**`dispatcher forecast-sync <yaml>`** — for each row in a terminal status (`Done`, `Blocked`, `Escalated`) whose `jira_key` is set, runs `forecast jira transition <jira_key> --to <target>` with an auto-generated comment:
- `Done` → includes `PR: <pr_url>`, iteration count, quality score
- `Blocked` → includes `blocked_reason`
- `Escalated` → "Escalated — needs human review" + `blocked_reason`

Read-only on the YAML.

**Smart detection.** Both subcommands check for `forecast` on PATH and `.forecast/config.yaml` in the project root (walking up). If either is missing, they print a soft-skip message and exit 0 — safe to chain `dispatcher run && dispatcher forecast-sync` even on machines without the forecast tool.

---

## Writing a new tasks YAML — checklist

When the Tasker is asked to write task rows for a forecast-managed project:

- [ ] Top-level `project:` matches the Jira project key (e.g., `SMG`, `FSG`)
- [ ] Top-level `epic:` set if all rows share an epic — saves repeating it per row
- [ ] Every row has `key`, `summary`, `description`, `type`, `labels` populated
- [ ] Every `labels` list includes exactly one `size:S|M|L|XL` entry
- [ ] Every `labels` list includes a `type:*` entry (per evenplay-mono CLAUDE.md convention)
- [ ] `key` is free-form but unique. Use semantic identifiers (`BSA-E2E-0-1`) or generic placeholders (`TBD-1`) — either works.
- [ ] `jira_key` is left empty unless the row corresponds to an existing Jira ticket.
- [ ] `blockedBy` references match `key` values (the dispatcher's local identifiers) elsewhere in the file
- [ ] Description is in block-scalar form (`description: |`) for multi-line content
- [ ] Optional fields (`priority`, `story_points`, `due_date`, etc.) added only when known

**Then run:**
```bash
dispatcher forecast-create tasks.yaml    # creates Jira tickets, rewrites placeholder keys
dispatcher run tasks.yaml --mode dry-run # validates the schema + shows the plan
dispatcher run tasks.yaml --mode unattended  # the actual run
dispatcher forecast-sync tasks.yaml      # transitions Jira to match dispatcher outcomes
```

---

## Common failure modes

| Failure | Cause | Fix |
|---------|-------|-----|
| `forecast-create` errors: "no .forecast/config.yaml" | Project hasn't been initialized | Run `forecast init` from the project root |
| `forecast-create` errors: "API auth failed" | Token expired or wrong project key | Check `~/.config/jira/credentials` or `$JIRA_API_TOKEN` |
| `forecast-create` skips a row unexpectedly | `jira_key` already set (intentionally or from a previous bridge run) | Check the row — clear `jira_key` if you really want a new ticket |
| `forecast-sync` skips a Done row | `jira_key` missing on that row | Run `forecast-create` first, or set `jira_key` by hand to an existing Jira ticket |
| Bridge overwrote my semantic key like `BSA-E2E-0-1` | Using a pre-fix version of the bridge | Fixed; the bridge now writes only to `jira_key` and leaves `key` alone |
| `forecast-sync` transitions fail with "no valid transition" | Source status doesn't allow this target (SMG workflow rules) | Check `forecast jira transitions <key>` for valid moves; adjust `status_mapping` |
| Dispatcher run fails: "task has no size: label" | Wrote labels as a string instead of a list | Use YAML list syntax: `labels: [size:M, area:schema]` |
| YAML write conflict between create + run | Bridge and dispatcher writing simultaneously | Both use the same FileLock — wait for create to finish before running |

---

## When NOT to use this schema

- **Single-task ad-hoc dispatch.** If you're just running one task by hand without an organized YAML, you don't need any of these fields — just write the inline Task Assignment for the Tasker.
- **Projects with no Jira / forecast integration.** Skip the optional forecast-mappable fields. The dispatcher works fine with just `key, summary, description, type, labels, blockedBy`.
- **Projects on a different ticket tracker.** Linear / GitHub Issues / etc. The bridge currently only talks to forecast (Jira). A new bridge module per tracker is the path forward; the YAML schema is intentionally tracker-agnostic for the dispatcher's purposes.
