---
name: spock-domain-analysis
description: >
  Domain analysis expertise for Spock. Activate when reading an existing codebase
  to understand its structure, identifying bounded contexts, mapping aggregates and
  entities, discovering existing patterns before planning a feature, investigating
  a bug or regression, analysing cross-cutting concerns, or when the task type is
  "Analysis / Investigation" with no immediate plan required.
---

# Spock — Domain Analysis Expertise

This skill gives Spock a structured methodology for reading and understanding
a codebase before planning. Apply it during Phase 1 (Interview) when gathering
codebase context, and at the start of Phase 2 (Plan) before proposing changes.

**Analysis precedes planning.** Never propose a plan based on assumptions about
the existing codebase. Always read first, then plan.

---

## Analysis Workflow

Execute these steps in order at the start of every planning session.
Skip a step only if it is provably irrelevant (document why in the plan context).

### Step 1 — Identify Architecture Style

```
search/codebase: "Features/" OR "Domain/" OR "Application/" OR "Infrastructure/"
```

| Finding                          | Architecture          | Next step                        |
|----------------------------------|-----------------------|----------------------------------|
| `Features/` at root              | Vertical Slice (VSA)  | Map feature folders → Step 2a   |
| `Domain/` + `Application/`       | Onion / Clean         | Map layers → Step 2b             |
| Controllers + Services folders   | Layered / MVC         | Note as tech debt, plan in VSA   |
| Mixture                          | Hybrid / In transition | Flag in plan, ask in interview   |

Load `spock-architecture` for detailed structure templates once architecture is identified.

### Step 2a — Map Vertical Slice Features

For each domain folder under `Features/`:

```
read/listDirectory: Features/{Domain}/
```

Produce a feature map:

```markdown
### Feature Map: {Domain}

| Feature           | Files present                              | Handler type  | Notes                    |
|-------------------|--------------------------------------------|---------------|--------------------------|
| CreateOrder       | Command, Handler, Endpoint                 | ICommandHandler | Missing Validator       |
| GetOrder          | Query, Handler, Response, Endpoint         | IQueryHandler |                          |
| UpdateOrder       | (none found)                               | —             | ⚠️ Not yet implemented   |
```

Flag missing files per feature (e.g. a Handler with no Endpoint, a Command with no Validator).

### Step 2b — Map Onion Layers

For each layer:

```
read/listDirectory: Domain/
read/listDirectory: Application/UseCases/
read/listDirectory: Infrastructure/Persistence/
```

Produce a layer summary:

```markdown
### Layer Map

**Domain**
- Entities: [list]
- Value Objects: [list]
- Domain Events: [list]
- Interfaces defined: [list — these are implemented in Infrastructure]

**Application**
- Commands: [list]
- Queries: [list]
- Interfaces consumed: [list — these come from Domain or Application/Interfaces]

**Infrastructure**
- EF Configurations: [list — one per entity]
- Repository implementations: [list]
- External service adapters: [list]

**Dependency violations found:** [list any Infrastructure → Domain reference going the wrong way]
```

### Step 3 — Identify Shared Abstractions

```
search/codebase: "ICommandHandler" OR "IQueryHandler" OR "Result<"
```

Record the canonical location of each:

| Abstraction        | File path                          | Implemented by (count) |
|--------------------|------------------------------------|------------------------|
| ICommandHandler    | Shared/Abstractions/...            | N handlers             |
| IQueryHandler      | Shared/Abstractions/...            | N handlers             |
| Result<T>          | Shared/Abstractions/...            | used by N handlers     |

If MediatR types (`IRequest`, `IRequestHandler`, `IMediator`) are found in any file,
flag them as ⚠️ TECH DEBT in the plan context. Do not plan new features using MediatR.

### Step 4 — Read the DbContext

```
search/codebase: "DbContext"
read/readFile: {path to AppDbContext.cs}
```

Produce an entity map:

```markdown
### Entity Map (from AppDbContext)

| DbSet              | Entity           | Key type | Has owned types | Notes              |
|--------------------|------------------|----------|-----------------|--------------------|
| Orders             | Order            | Guid     | Yes (Address)   |                    |
| Customers          | Customer         | Guid     | No              |                    |
| Products           | Product          | Guid     | No              | ⚠️ No config found |
```

Flag any entity without a corresponding `IEntityTypeConfiguration<T>` file.

### Step 5 — Identify Integration Points

```
search/codebase: "HttpClient" OR "IHttpClientFactory" OR "GramJS" OR "Telegram"
search/codebase: "AzureDevOps" OR "azure-devops-cli"
```

Record each integration point:

| Integration        | Location                 | Pattern used         | Notes                        |
|--------------------|--------------------------|----------------------|------------------------------|
| Telegram MTProto   | Infrastructure/Telegram/ | GramJS wrapper       | Node.js service              |
| Azure DevOps API   | Infrastructure/AzDo/     | HttpClient typed     |                              |
| External API X     | Infrastructure/ExternalX | HttpClient typed     | ⚠️ No retry policy found     |

### Step 6 — Identify Test Coverage

```
search/fileSearch: "*Tests.cs" OR "*Spec.ts" OR "*spec.ts"
read/listDirectory: tests/
```

Produce a coverage summary:

```markdown
### Test Coverage

| Feature / Component        | Has unit tests | Has integration tests | Notes                  |
|----------------------------|----------------|-----------------------|------------------------|
| CreateOrder handler        | ✅             | ❌                    |                        |
| GetOrder handler           | ❌             | ❌                    | ⚠️ No tests found      |
| AppDbContext configuration | —              | ✅                    |                        |
```

Flag any handler modified by the current plan that has zero test coverage.
Load `spock-risk-assessment` to score the coverage gap as a risk.

---

## Domain Concept Extraction

When the task involves a new or changed business concept, extract domain language
from the existing codebase before the Interview phase. This prevents proposing
names that conflict with established terminology.

### Ubiquitous Language Scan

```
search/codebase: "{concept keyword}"  // e.g. "Order", "Customer", "Invoice"
```

Record all usages:

```markdown
### Ubiquitous Language: Order

| Term           | Used in                          | Meaning in codebase                  |
|----------------|----------------------------------|--------------------------------------|
| Order          | Order.cs, OrderStatus.cs         | A customer purchase                  |
| OrderItem      | OrderItem.cs                     | Line item within an Order            |
| OrderSummary   | GetOrderResponse.cs              | Read-only projection, not the entity |
| CreateOrder    | Features/Orders/CreateOrder/     | The act of placing a new Order       |
```

Propose new names that are consistent with this language. If a concept does not
exist yet, propose a name and flag it as `PROPOSED TERM — confirm in interview`.

### Bounded Context Signals

Look for these patterns to infer context boundaries:

| Signal                                               | Inference                                             |
|------------------------------------------------------|-------------------------------------------------------|
| Two features reference the same entity differently   | Possible bounded context boundary                     |
| DTO has same name as entity but different fields     | Context translation happening — map carefully         |
| Feature imports from a different domain folder       | Cross-context coupling — flag as risk                 |
| Separate DbContext classes                           | Explicit context separation                           |
| `{Domain}ModuleExtensions.cs` per domain folder      | Intentional VSA context separation — respect it       |

---

## Reading Patterns — Quick Searches

Use these `search/codebase` queries to quickly orient in an unfamiliar area.

### Find all commands and queries

```
search/codebase: ": ICommandHandler<"
search/codebase: ": IQueryHandler<"
```

### Find all endpoints

```
search/codebase: "MapPost\|MapGet\|MapPut\|MapDelete\|MapPatch"
```

### Find all domain events

```
search/codebase: "DomainEvent\|IDomainEvent\|INotification"
```

### Find all validators

```
search/codebase: "AbstractValidator<\|IOrderValidator\|IValidator<"
```

### Find all EF configurations

```
search/fileSearch: "*Configuration.cs"
search/codebase: "IEntityTypeConfiguration"
```

### Find error handling patterns

```
search/codebase: "Result.Failure\|Result.Success\|Result.NotFound"
search/codebase: "catch (Exception\|ProblemDetails\|UseExceptionHandler"
```

---

## Analysis Output Format

When the scenario is "Analysis / Investigation" (no plan needed), produce this output:

```markdown
## Analysis: {Subject}

### Architecture
- Style: [VSA | Onion | Layered | Hybrid]
- Status: [Clean | Has tech debt — describe]

### Scope of Change
- Domains affected: [list]
- Layers affected: [list — for Onion]
- Shared abstractions touched: [list or "none"]
- DB schema changes: [Yes — describe | No]

### Existing Patterns Found
- [Pattern 1 — location — how it works]
- [Pattern 2 — location — how it works]

### Gaps and Anomalies
- ⚠️ [Gap 1 — description — recommendation]
- ⚠️ [Gap 2 — description — recommendation]

### Integration Points Identified
- [Integration — location — notes]

### Recommendation
[1-3 sentences: what the analysis reveals and what the next step should be]
```

---

## References

- [codebase-reading-guide.md](./references/codebase-reading-guide.md) — Annotated reading order for unfamiliar C# and TypeScript projects
- [domain-patterns-glossary.md](./references/domain-patterns-glossary.md) — Canonical names for DDD patterns as used in this codebase
