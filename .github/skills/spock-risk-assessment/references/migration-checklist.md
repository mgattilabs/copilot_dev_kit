# Migration Checklist

Step-by-step checklist for Spock to include in any plan that modifies the database schema.
Copy the relevant sections into the plan under the migration phase.

---

## Pre-Migration (include in Plan steps)

- [ ] Migration type identified: Additive / Destructive / Data-backfill
- [ ] Migration reversibility confirmed: Reversible / Non-reversible (with reason)
- [ ] Estimated rows affected stated in plan (or flagged as unknown)
- [ ] Downtime requirement stated: Zero-downtime / Maintenance window required
- [ ] Migration script generated: `dotnet ef migrations add {MigrationName}`
- [ ] Migration reviewed by Spock: up/down methods both inspected
- [ ] Tested on dev DB: `dotnet ef database update`
- [ ] Tested on staging DB (copy of prod schema): manual step flagged in plan

## Deployment (include as acceptance criteria)

- [ ] Migration runs without errors in staging
- [ ] Rollback tested: `dotnet ef database update {PreviousMigration}` succeeds
- [ ] Application smoke test after migration: key endpoints respond correctly
- [ ] If data-backfill: row counts verified before and after

## Post-Migration (flag in Follow-up / Tech Debt if applicable)

- [ ] Old column/table removal scheduled (for two-step destructive migrations)
- [ ] Monitoring alert added if new index changes query plans significantly
- [ ] Documentation updated in Alexandria's `IMPLEMENTATION-LOG.md`

---

## Migration Naming Convention

```
{timestamp}_{Verb}{Subject}
```

Examples:
- `20250223_AddStatusColumnToOrders`
- `20250223_CreateProductsTable`
- `20250223_RenameCustomerEmailToContactEmail`
- `20250223_BackfillOrderTotals`

Verb options: `Add`, `Remove`, `Create`, `Drop`, `Rename`, `Backfill`, `AlterType`

---

## Danger Signals — Flag These as 🔴 Critical

Any migration that meets one of these conditions must be flagged as Critical
and requires explicit approval before Neo starts:

- Drops a column or table that has existing production data
- Changes a column type in a way that loses precision (e.g., `decimal(10,2)` → `int`)
- Adds a NOT NULL column with no default value and no backfill script
- Removes a FK constraint that was enforcing referential integrity
- Has no rollback path (EF Core `Down()` method is empty or throws)
