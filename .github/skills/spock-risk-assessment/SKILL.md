---
name: spock-risk-assessment
description: >
  Risk assessment expertise for Spock. Activate when evaluating risks in any
  implementation plan, estimating complexity and impact of breaking changes,
  assessing database migrations, identifying cross-feature coupling, reviewing
  plans involving external APIs (Azure, Telegram MTProto), scoring effort for
  Azure DevOps WorkItems, or when producing the Risk Assessment section of any
  plan document.
---

# Spock — Risk Assessment Expertise

This skill gives Spock a structured, repeatable methodology for identifying,
scoring, and mitigating risks in implementation plans. Apply it during Phase 2
(Plan) when building the Risk Assessment table, and during Phase 1 (Interview)
to formulate targeted questions about high-risk areas.

**Risks are not optional.** Every plan must include a Risk Assessment section,
even if all risks are Low. An empty risk table signals incomplete analysis.

---

## Risk Scoring System

Score each risk on two axes and derive a **Priority**:

| Impact | Probability | Priority |
|--------|-------------|----------|
| High   | High        | 🔴 Critical — block plan approval, resolve before starting |
| High   | Med         | 🟠 High — mitigation required in the plan                  |
| High   | Low         | 🟡 Medium — mitigation recommended, monitor during impl     |
| Med    | High        | 🟠 High — mitigation required                              |
| Med    | Med         | 🟡 Medium — note and monitor                               |
| Med    | Low         | 🟢 Low — document, no action required                      |
| Low    | Any         | 🟢 Low — document only                                     |

**Rule:** A plan with any 🔴 Critical risk must not proceed to Neo until the risk
is resolved or explicitly accepted by Michele with a written rationale in the plan.

---

## Risk Catalogue by Category

Use this catalogue to systematically check risks. Work through every category
for every plan — skip a category only if it is provably irrelevant.

### 1. Breaking Changes

Triggered by: API contract changes, renamed/removed public interfaces, DB column
changes, enum value changes, event schema changes.

| Risk                                      | Default Impact | Probe question for Interview                           |
|-------------------------------------------|----------------|--------------------------------------------------------|
| REST endpoint URL or method changed       | High           | Are external clients calling this endpoint?            |
| Request/Response DTO field removed        | High           | Is the DTO consumed by other features or frontends?    |
| Enum value renamed or removed             | Med            | Is this enum persisted to DB or sent over the wire?    |
| Public interface method signature changed | High           | How many implementations/mocks exist for this iface?   |
| Event schema changed                      | High           | Are there consumers of this event outside this service?|

### 2. Database & Migrations

Triggered by: new tables, column additions/removals/renames, index changes,
FK additions, data type changes, backfill requirements.

| Risk                                       | Default Impact | Probe question for Interview                              |
|--------------------------------------------|----------------|-----------------------------------------------------------|
| Column rename on a table with existing data| High           | How many rows are in this table in production?            |
| NOT NULL column added without default      | High           | Will a migration run without downtime or a backfill step? |
| FK added to existing table                 | Med            | Are there orphan rows that would violate the constraint?  |
| Index removed                              | Med            | Is this index used by any critical query path?            |
| Enum stored as int changed to string       | High           | Are stored values compatible with the new representation? |

**Migration rule:** Any plan that changes the DB schema must include an explicit
migration step with this structure:

```
MIGRATION STEP
  File: Migrations/{timestamp}_{MigrationName}.cs
  Type: [Additive | Destructive | Data-backfill]
  Reversible: [Yes | No — explain why not]
  Estimated rows affected: [count or "unknown — flag for prod check"]
  Downtime required: [Yes | No]
```

Flag Destructive or non-reversible migrations as 🔴 Critical risk by default.

### 3. Cross-Feature Coupling

Triggered by: modifying shared abstractions (`ICommandHandler`, `Result<T>`,
`AppDbContext`), modifying shared infrastructure, touching a `{Domain}Module.cs`
used by multiple features.

| Risk                                            | Default Impact | Probe question for Interview                              |
|-------------------------------------------------|----------------|-----------------------------------------------------------|
| Shared abstraction interface modified           | High           | How many handlers implement this interface?               |
| AppDbContext entity configuration changed       | Med            | Which features query this entity?                         |
| Shared DTO or response model changed            | Med            | Which endpoints return this model?                        |
| Domain module DI registration changed           | Med            | Which features are registered in this module?             |

**Detection method:** Run `search/usages` on the affected file/interface before
finalising the plan. Never estimate coupling from memory.

### 4. External Dependencies

Triggered by: any call to an external service (Azure APIs, Telegram MTProto,
third-party REST APIs, email services, payment providers).

| Risk                                            | Default Impact | Standard mitigation to plan                              |
|-------------------------------------------------|----------------|----------------------------------------------------------|
| External API unavailable                        | Med–High       | Plan a circuit breaker or retry policy                   |
| External API rate-limited                       | Med            | Plan exponential backoff + jitter                        |
| Telegram MTProto requires GramJS wrapper        | High           | Cannot use direct HTTP — plan GramJS service layer       |
| Azure service quota exceeded                    | Med            | Plan quota check + alerting                              |
| Third-party API contract changed                | Med            | Plan version pinning + contract tests                    |

**MTProto-specific:** Telegram's MTProto protocol cannot be called with plain
HTTP/curl. Always plan a GramJS (Node.js) wrapper service for Telegram integration.
Flag any plan that attempts direct HTTP calls to Telegram as 🔴 Critical.

### 5. Performance

Triggered by: queries on large tables, loading collections without projection,
synchronous operations on hot paths, missing pagination.

| Risk                                           | Default Impact | Probe question for Interview                             |
|------------------------------------------------|----------------|----------------------------------------------------------|
| Query loads full entity for read-only ops      | Med            | Is AsNoTracking + projection planned?                    |
| Collection loaded without pagination           | Med–High       | What is the expected max row count?                      |
| N+1 query (loop + DB call)                     | High           | Is Include() or a single JOIN query planned?             |
| Synchronous I/O on a hot path                  | High           | Is async/await used throughout the call chain?           |
| Missing index on a filtered/sorted column      | Med            | Is a migration adding the index included in the plan?    |

### 6. Security

Triggered by: new endpoints, new data fields, changes to authentication or
authorization logic, logging of user data.

| Risk                                            | Default Impact | Standard check                                           |
|-------------------------------------------------|----------------|----------------------------------------------------------|
| New endpoint without authorization attribute    | High           | Flag immediately — all endpoints require auth by default |
| Sensitive data returned in response DTO         | High           | PII, tokens, passwords must never appear in responses    |
| User input written to logs                      | Med            | Sanitise before logging                                  |
| CORS policy changed                             | Med            | Document allowed origins explicitly in the plan          |
| JWT scope not checked on new endpoint           | High           | Plan explicit scope validation                           |

### 7. Test Coverage Gaps

Triggered by: complex business logic, edge cases in handlers, external service calls.

| Risk                                            | Default Impact | Standard mitigation                                      |
|-------------------------------------------------|----------------|----------------------------------------------------------|
| Handler with multiple branching paths, no tests | Med            | Plan unit tests for each branch (happy + failure)        |
| Validator not tested                            | Med            | Plan validator unit tests with valid/invalid inputs      |
| External service call not mocked in tests       | Med            | Plan NSubstitute mock + test for failure scenario        |
| Migration not tested on copy of prod data       | High           | Flag for manual pre-deploy verification                  |

---

## Risk Assessment Table Format

Every plan's Risk Assessment section must use this exact format:

```markdown
## Risk Assessment

| Risk | Category | Impact | Probability | Priority | Mitigation |
|------|----------|--------|-------------|----------|------------|
| [description] | [category] | High/Med/Low | High/Med/Low | 🔴/🟠/🟡/🟢 | [concrete action] |
```

Mitigations must be **concrete actions**, not vague reassurances.

✅ Good: "Add `AsNoTracking()` + projection in `GetOrderHandler` — Neo to verify"
❌ Bad: "Be careful with queries"

✅ Good: "Run migration on a staging DB copy before prod deploy — Alexandria to flag"
❌ Bad: "Test the migration"

---

## Risk-Driven Interview Questions

Use these during Phase 1 (Interview) when the scenario suggests elevated risk.
Append them to the relevant question category in the interview output.

### When the task involves DB changes

- Will this migration run against a table with existing data?
- Is downtime acceptable during the migration, or must it be zero-downtime?
- Are there any consumers (reports, external systems) reading this table directly?

### When the task involves modifying a shared abstraction

- How many features currently implement or depend on this interface?
- Is there a test suite that would catch regressions across all consumers?

### When the task involves a new endpoint

- Should this endpoint require authentication? What role or scope?
- Will this endpoint be called by an external client or only by our frontend?

### When the task involves an external API

- Is there a sandbox/test environment available for this API?
- What is the API's SLA and rate limit policy?
- Do we have error handling patterns for this API already in the codebase?

---

## Complexity Modifiers

Start with the base complexity from `spock-architecture` (S/M/L matrix),
then apply these modifiers:

| Condition present                                | Modifier        |
|--------------------------------------------------|-----------------|
| Any 🔴 Critical risk                             | Escalate one level (S→M, M→L) |
| Destructive or non-reversible migration          | Escalate one level |
| Cross-feature coupling to 3+ other features      | Escalate one level |
| External API with no sandbox                     | +1 risk item, Med impact |
| No existing tests in the affected area           | +1 risk item, Med impact |

---

## References

- [risk-mitigation-patterns.md](./references/risk-mitigation-patterns.md) — Standard mitigations per risk type with implementation sketches
- [migration-checklist.md](./references/migration-checklist.md) — Step-by-step pre/post-deploy checklist for DB migrations
