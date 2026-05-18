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
  - key: TBD-1                              # placeholder until bridge creates the Jira ticket
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
| `key` | yes (placeholder ok) | (output — Jira assigns) | Issue key |
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

## How the bridge uses these fields

**`dispatcher forecast-create <yaml>`** — for each task row whose `key` matches the placeholder pattern, runs `forecast jira create` with the mapped flags. On success, captures `Created: KEY-NN` from stdout and writes the real Jira key back to the YAML row.

Idempotent. Re-running after a partial failure picks up only the still-placeholder rows.

**`dispatcher forecast-sync <yaml>`** — for each row in a terminal status (`Done`, `Blocked`, `Escalated`) with a real Jira key, runs `forecast jira transition <key> --to <target>` with an auto-generated comment:
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
- [ ] Keys start as `TBD-1, TBD-2, ...` for new tickets (or whatever `placeholder_prefix` you've set)
- [ ] `blockedBy` references match real or placeholder keys that exist elsewhere in the file
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
| `forecast-create` skips a row unexpectedly | Key already looks like a real Jira key | Use `TBD-` prefix (or change `forecast.placeholder_prefix` in the YAML) |
| `forecast-sync` transitions fail with "no valid transition" | Source status doesn't allow this target (SMG workflow rules) | Check `forecast jira transitions <key>` for valid moves; adjust `status_mapping` |
| Dispatcher run fails: "task has no size: label" | Wrote labels as a string instead of a list | Use YAML list syntax: `labels: [size:M, area:schema]` |
| YAML write conflict between create + run | Bridge and dispatcher writing simultaneously | Both use the same FileLock — wait for create to finish before running |

---

## When NOT to use this schema

- **Single-task ad-hoc dispatch.** If you're just running one task by hand without an organized YAML, you don't need any of these fields — just write the inline Task Assignment for the Tasker.
- **Projects with no Jira / forecast integration.** Skip the optional forecast-mappable fields. The dispatcher works fine with just `key, summary, description, type, labels, blockedBy`.
- **Projects on a different ticket tracker.** Linear / GitHub Issues / etc. The bridge currently only talks to forecast (Jira). A new bridge module per tracker is the path forward; the YAML schema is intentionally tracker-agnostic for the dispatcher's purposes.
