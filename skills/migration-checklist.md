---
name: migration-checklist
description: Schema migration checklist for the Tasker to enforce on any task touching .sql, .prisma, or migration files. Covers idempotency, FK indexes, NOT NULL splits, lock windows, primary key choice, and naming convention.
---

# Migration Checklist

Load this skill when the task touches schema migrations. Add the checklist below to the Task Assignment's Definition of Done, and verify each item in the Completion Report.

---

## Filename Convention

New migration files use a wall-clock UTC timestamp prefix:

```
YYYYMMDDHHMM_short_description.up.sql
YYYYMMDDHHMM_short_description.down.sql
```

Generated with `date -u +"%Y%m%d%H%M"` at the moment the file is created. **Pick a fresh timestamp** per migration — do NOT reuse a prefix from another in-flight migration in the same folder, even if shipping together. Existing files retain their numeric `000NNN_` prefix; new ones use the timestamp form.

---

## Primary-Key Type (new tables)

Rule, in priority order:

1. **Surrogate for a cross-service-created object → UUID.** If the row represents a thing created by one microservice in response to a request from another, the originating service generates a UUID client-side and sends it with the create RPC. Retry-safe: the caller knows the ID before the RPC succeeds. The `INSERT` uses `ON CONFLICT (id) DO NOTHING` so duplicate creates from retries return the existing row.
2. **Internal monotonic sequence consumed only by the producing service → BIGSERIAL.** Use when the column is a counter (not an identity) and needs ordered comparison. UUID v4 doesn't sort; UUID v7 lacks Go-ecosystem support.
3. **FK to an existing BIGINT parent → match the parent.** Carry the type debt as a coordinated-migration item; do not create mixed-type FK chains incrementally.
4. **Natural composite key available → use it, no surrogate.** Use when the row is uniquely identified by a domain-meaningful tuple.

---

## Pre-Dispatch Checklist (Tasker adds to Task Assignment)

```markdown
### Migration Checklist (MUST complete before claiming done)
- [ ] Filename uses fresh UTC timestamp prefix `YYYYMMDDHHMM_description.up.sql` / `.down.sql`
- [ ] PK type follows the priority rule above
- [ ] Up migration is idempotent (`IF NOT EXISTS`, `ON CONFLICT DO NOTHING`, etc.)
- [ ] Down migration exactly reverses up — mentally walk through: up → down = no net change
- [ ] No full-table rewrite on a large table without a comment noting the lock window
- [ ] Every new foreign key column has a corresponding index
- [ ] New `NOT NULL` columns have a `DEFAULT` value, OR the migration is split: add nullable → backfill data → add constraint
- [ ] If the table will be created cross-service (UUID PK case): create RPC accepts client-supplied UUID; SQL uses `ON CONFLICT (id) DO NOTHING`
- [ ] Tested locally: `up` runs clean, `down` runs clean, `up` again runs clean
```

---

## Tasker Verification (on Completion Report)

Verify on the Coder's submission:

- **Filename matches the convention** — reject if the prefix is reused, not UTC-current, or in the legacy `000NNN_` form for a new file.
- **Down migration was actually tested** — Completion Report must include test output for `up → down → up` cycle.
- **FK indexes exist** — grep the migration for `FOREIGN KEY` and verify a matching `CREATE INDEX` (or `INDEX` inline) exists for each.
- **NOT NULL constraints on existing tables** — if the migration adds `NOT NULL` to a column on an existing table, verify either a `DEFAULT` clause or a three-phase split (add nullable → backfill → constrain). A bare `NOT NULL` on an existing column is a lock + table-rewrite hazard.
- **Cross-service-created tables (UUID PK)** — check that the create RPC accepts a client-supplied UUID and the SQL uses `ON CONFLICT (id) DO NOTHING`. A `RETURNING id` without conflict handling fails retry-safety.
- **No `DROP COLUMN` / `DROP TABLE` without a deprecation path** — schema removals on shared tables need an explicit comment naming the consumers that have been migrated off. If the comment isn't there, escalate to human.
- **`down` migration handles data created by `up`** — if `up` populates a table, `down` truncates it (or the migration documents why preservation is intended).

---

## Common Failure Modes

| Failure | Cause | Fix |
|---------|-------|-----|
| `up` then `up` errors | Missing `IF NOT EXISTS` on `CREATE TABLE` / `CREATE INDEX` | Add `IF NOT EXISTS` |
| `down` then `up` errors | `down` didn't drop everything `up` added | Audit each `up` statement has a `down` reversal |
| FK query slow | Missing FK index | Add `CREATE INDEX ... ON child(parent_id)` |
| `NOT NULL` migration deadlocks prod | Full-table rewrite under exclusive lock | Split: nullable column → backfill in batches → `ALTER ... SET NOT NULL` |
| Duplicate row on retry | Cross-service create without `ON CONFLICT` | Add `ON CONFLICT (id) DO NOTHING` and have caller supply UUID |
| Mixed-type FK chain | New table picked UUID while parent is BIGINT | Match parent type; queue UUID migration of both for coordinated landing |

---

## Definition of Done Addition

When loading this skill, the Tasker appends to the Coder's Definition of Done:

- [ ] All Migration Checklist items above are complete
- [ ] `up → down → up` cycle output included in Completion Report
- [ ] FK indexes present for every new FK column
- [ ] PK type choice documented in the migration file's header comment (which rule applied and why)
