# Risk Mitigation Patterns

Standard mitigation approaches for each risk category. These are planning
sketches — not code. Reference them when writing the Mitigation column of the
Risk Assessment table and when defining plan steps for risk reduction.

---

## Breaking Changes — Mitigations

### API Contract Change

**Preferred pattern: versioned endpoint**

Plan steps:
1. Keep the existing endpoint untouched (`/api/v1/orders`)
2. Create a new versioned endpoint (`/api/v2/orders`) with the new contract
3. Mark the old endpoint as deprecated in OpenAPI docs (`.WithSummary("[DEPRECATED]...")`)
4. Plan a removal phase in a future plan (flag with `// TODO: remove after <date>`)

**When to use instead: consumer-controlled migration**

If the only consumer is our own frontend, plan a coordinated deploy:
- Step 1: Deploy backend with both old + new endpoint
- Step 2: Deploy frontend pointing to new endpoint
- Step 3: Deploy backend removing old endpoint
- Flag the coordinated deploy requirement in the plan's Testing Strategy

### DTO Field Removal

Never remove a field in a single deploy. Plan a two-step approach:
1. Mark field as `[Obsolete]` in C# or add comment in TypeScript DTO
2. Verify no consumer reads the field (search/usages before proceeding)
3. Remove the field in a subsequent plan

---

## Database Migrations — Mitigations

### Non-destructive additive migration (safe)

Plan structure:
```
Phase X: Database Migration
  Steps:
  1. CREATE: Migrations/{timestamp}_Add{ColumnName}To{Table}.cs
  2. Column added as nullable OR with default value
  3. Run: dotnet ef migrations add ... && dotnet ef database update
  Acceptance Criteria:
  - [ ] Migration runs without errors on dev DB
  - [ ] Migration runs without errors on a copy of prod data (manual step)
  - [ ] Rollback verified: dotnet ef database update {PreviousMigration}
```

### Destructive migration (column rename/removal)

Plan structure:
```
Phase X: Database Migration — ⚠️ DESTRUCTIVE
  Steps:
  1. Add new column with correct name (nullable)
  2. Backfill data: UPDATE statement in a data migration script
  3. Change application code to use new column
  4. Deploy and verify data integrity
  5. Drop old column in a SEPARATE subsequent migration (separate deploy)
  Acceptance Criteria:
  - [ ] Backfill script validated on staging
  - [ ] Old column removal scheduled for next sprint
  - [ ] Rollback plan documented
```

### Zero-downtime migration

For tables with high traffic, plan the expand/contract pattern:
1. **Expand**: add new column, keep old — both columns written by app
2. **Migrate**: backfill old → new asynchronously
3. **Contract**: switch reads to new column, stop writing old
4. **Cleanup**: drop old column (separate deploy)

Flag any migration that cannot follow this pattern as requiring maintenance window.

---

## Cross-Feature Coupling — Mitigations

### Shared interface modification

Before proposing a change to `ICommandHandler`, `IQueryHandler`, or `Result<T>`:
1. Plan a `search/usages` step to enumerate all implementations
2. List every affected file in the plan under "Files to Modify"
3. If more than 5 files are affected, consider an additive approach:
   - Add a new interface method with a default implementation
   - Migrate implementations incrementally rather than all-at-once

### AppDbContext entity configuration change

1. Plan the configuration change in `Configurations/{Entity}Configuration.cs`
2. List all features that use this entity (`search/usages` on the DbSet property)
3. Plan test execution for all affected feature handlers

---

## External Dependencies — Mitigations

### Resilience for external API calls

Plan the following wrapper when any external API is introduced:

```
{ExternalService}Client
  - RetryPolicy: 3 attempts, exponential backoff (1s, 2s, 4s), jitter
  - CircuitBreaker: open after 5 failures in 30s, half-open after 60s
  - Timeout: set per API SLA (never rely on default HttpClient timeout)
  - Logging: log attempt, success, and failure with structured fields
```

Register as `IHttpClientFactory`-based typed client in DI. Plan the
registration in the relevant `{Domain}Module.cs`.

### Telegram MTProto via GramJS

Never plan direct HTTP calls to `api.telegram.org` for MTProto operations.
Plan a GramJS wrapper service:

```
TelegramGramJsService (Node.js microservice)
  - Exposes HTTP endpoints for the .NET backend to consume
  - Handles session management and MTProto protocol internally
  - .NET backend calls it via HttpClient with retry policy
```

For Bot API only (not MTProto), plain HTTP is acceptable.

### Azure service quota / throttling

Plan a quota check step before any new Azure resource usage:
1. Document the quota limit in the plan (check Azure portal or docs)
2. Plan an alert if usage approaches 80% of quota
3. If quota increase is needed, flag as a pre-implementation dependency

---

## Performance — Mitigations

### N+1 query

Plan either:
- `Include()` for simple cases (small collections, simple joins)
- Single `SELECT` with `Join` and projection for large collections
- Split query pattern for very large related collections: `.AsSplitQuery()`

Always state the expected max collection size in the plan. If unknown, add it
as an interview question.

### Missing pagination

Plan a standard pagination response wrapper for any collection endpoint:

```
PagedResponse<T>
  Items: List<T>
  PageNumber: int
  PageSize: int
  TotalCount: int
  HasNextPage: bool
```

Default page size: 20. Maximum page size: 100. Enforce in the handler.

### Missing index

Plan index additions in the same migration as the feature:
- Index on FK columns (EF Core does not always add these automatically)
- Composite index for filtered + sorted queries (most selective column first)
- Flag index drops as requiring impact analysis via `search/usages` on the query

---

## Security — Mitigations

### New endpoint without authorization

All Minimal API endpoints must include one of:
- `.RequireAuthorization()` — requires authenticated user
- `.RequireAuthorization("PolicyName")` — requires specific policy
- `.AllowAnonymous()` with explicit justification in the plan

Plan the authorization requirement before Neo starts. Unauthenticated endpoints
are 🔴 Critical and block plan approval.

### PII in response DTOs

Plan a DTO audit step for any new response object that touches user data:
1. List every field in the DTO
2. Mark fields as PII or non-PII
3. Remove or mask PII fields not required by the client
4. Add a comment in the plan: "VERIFIED: no PII exposed — [field list]"

---

## Test Coverage — Mitigations

### Handler with no tests

Plan test cases with this structure per handler:

```
{Feature}HandlerTests
  Handle_ValidCommand_Returns{Expected}       // happy path
  Handle_Invalid{Condition}_ReturnsFailure   // business rule violation
  Handle_{EntityName}NotFound_ReturnsNotFound // not-found case
  Handle_DbException_Propagates              // infrastructure failure
```

Minimum: happy path + one failure case. Aim for full branch coverage on handlers
with complex business logic.

### Validator not tested

Plan validator tests as:
```
{Feature}ValidatorTests
  Validate_Valid{Feature}_PassesValidation
  Validate_Missing{RequiredField}_FailsValidation
  Validate_Invalid{Field}_FailsWithCorrectMessage
```
