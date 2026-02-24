---
name: spock-architecture
description: >
  Architecture planning expertise for Spock. Activate when planning .NET/C# features,
  choosing between Vertical Slice and Onion/Clean Architecture, designing CQRS without
  MediatR, evaluating EF Core patterns, planning Azure DevOps pipelines, assessing
  complexity of Angular or React frontends, or any task requiring an architectural
  decision on the current tech stack (C#, Python, TypeScript).
---

# Spock — Architecture Planning Expertise

This skill provides Spock with architectural decision frameworks, patterns, and
constraints specific to this project. Use it to produce accurate, opinionated plans
that align with the team's standards and tooling choices.

**You NEVER write code.** Use this skill to plan WHAT to build and WHY, not HOW.

---

## Tech Stack Constraints

| Layer        | Technology                          | Notes                                      |
|--------------|-------------------------------------|--------------------------------------------|
| Backend      | C# / .NET (latest LTS)             | C# 14, ASP.NET Core Minimal API            |
| ORM          | Entity Framework Core 9             | SQL Server or SQLite in dev                |
| CQRS         | **Manual — no MediatR**             | MediatR became commercial at v12+          |
| Validation   | DataAnnotations or manual           | FluentValidation only if already in project|
| Testing      | xUnit + NSubstitute + FluentAssertions |                                         |
| Frontend     | Angular 21 or React                 | Standalone components, Signals (Angular)   |
| Pipelines    | Azure DevOps YAML                   |                                            |
| AI/ML        | Python, DSPy                        |                                            |

> ⚠️ **MediatR is prohibited.** Always plan CQRS using manual interfaces and handlers.
> See [cqrs-without-mediatR.md](./references/cqrs-without-mediatR.md) for the approved pattern.

---

## Architecture Decision: Vertical Slice vs Onion

### When to recommend Vertical Slice (default)

- Feature-based development (one team, one feature at a time)
- CRUD-heavy or medium-complexity business logic
- New greenfield projects
- Speed of delivery is a priority

```
Features/
  Orders/
    CreateOrder/
      CreateOrderCommand.cs       // Record with input data
      CreateOrderHandler.cs       // Implements ICommandHandler<CreateOrderCommand, Result<Guid>>
      CreateOrderValidator.cs     // Manual validation logic or DataAnnotations
      CreateOrderEndpoint.cs      // Minimal API registration
    GetOrder/
      GetOrderQuery.cs
      GetOrderHandler.cs
      GetOrderResponse.cs
  Customers/
    ...
Shared/
  Abstractions/
    ICommandHandler.cs
    IQueryHandler.cs
    Result.cs
  Infrastructure/
    AppDbContext.cs
```

### When to recommend Onion / Clean Architecture

- Complex business rules requiring strict layer isolation
- Domain logic that must be tested independently of infrastructure
- Multiple delivery mechanisms (API + Worker + CLI)
- Long-lived projects where domain stability is critical

```
Domain/          // Entities, Value Objects, Domain Events, Interfaces — ZERO dependencies
Application/     // Use Cases (Commands/Queries), ICommandHandler, IQueryHandler
Infrastructure/  // EF Core, external APIs, implementations of Application interfaces
WebApi/          // Minimal API endpoints, DI registration, middleware
```

**Dependency rule (absolute):** arrows point inward. `Infrastructure → Application → Domain`.
Domain never references any outer layer. Violating this is a blocker in plan review.

---

## Manual CQRS Pattern (No MediatR)

Plan features using the following interfaces. Neo will implement them — your job
is to identify which commands and queries are needed and which files to create/modify.

### Interfaces (live in `Shared/Abstractions/`)

```
ICommandHandler<TCommand, TResult>   // For write operations
IQueryHandler<TQuery, TResult>       // For read operations
```

### File naming convention for a new feature

| File                         | Purpose                                          |
|------------------------------|--------------------------------------------------|
| `{Feature}Command.cs`        | Input record for write operations                |
| `{Feature}Handler.cs`        | Implements `ICommandHandler`                     |
| `{Feature}Query.cs`          | Input record for read operations                 |
| `{Feature}QueryHandler.cs`   | Implements `IQueryHandler`                       |
| `{Feature}Response.cs`       | DTO returned by queries                          |
| `{Feature}Endpoint.cs`       | Minimal API — injects handler directly via DI    |
| `{Feature}Validator.cs`      | Optional — manual validation or DataAnnotations  |

> Handlers are registered in DI and injected directly into endpoints.
> No pipeline, no bus, no mediator. Simple, explicit, testable.

---

## Result Pattern

All handlers return `Result<T>` (never throw for business logic).
Plan error paths explicitly — identify success, validation failure, and not-found cases.

```
Result<T>
  .Success(value)      // happy path
  .Failure(error)      // business rule violation
  .NotFound(id)        // entity not found (optional variant)
```

Endpoints map `Result<T>` to HTTP status codes. Plan this mapping per feature.

---

## EF Core Planning Rules

When planning data access, specify:

1. **Read vs Write context**: reads use `AsNoTracking()` + projections, writes use tracked entities
2. **Migration needed?** Flag any schema change as a required migration step in the plan
3. **N+1 risk?** If the feature loads collections, flag `Include()` or projection requirement
4. **Concurrency?** If multiple writers are possible, flag optimistic concurrency (`RowVersion`)

---

## Complexity Assessment Matrix

Use this to assign S/M/L per plan phase:

| Label | Criteria                                                                 |
|-------|--------------------------------------------------------------------------|
| **S** | Single file change, no new abstractions, no DB migration, clear scope    |
| **M** | 3–6 files, one new abstraction or migration, some integration points     |
| **L** | 7+ files, new layer or module, DB schema change, cross-feature impact    |

Flag **L** phases for explicit approval before Neo starts implementation.

---

## Clean Code Constraints for Planning

These rules apply to all plans — ensure they are reflected in file assignments and steps:

- **Single Responsibility**: every handler does one thing. If a handler needs to call two
  external services and update two aggregates, split it.
- **Early Return**: plan validation steps before the happy path. Order matters in the plan.
- **No Magic Numbers**: flag any hardcoded threshold (timeouts, limits, sizes) as a named constant
  to define. List them in the plan under a "Constants to define" section.
- **Expressive Conditions**: if a business rule has a complex condition, plan a dedicated
  private method or named variable for it (name it in the plan, Neo will implement it).
- **Comments only for WHY**: do not plan code comments that describe what the code does.

---

## Naming Conventions Summary

| Element            | C#              | Python           | TypeScript                     |
|--------------------|-----------------|------------------|--------------------------------|
| Variables          | `camelCase`     | `snake_case`     | `camelCase`                    |
| Constants          | `PascalCase`    | `UPPER_SNAKE`    | `UPPER_SNAKE`                  |
| Methods/Functions  | `PascalCase`    | `snake_case`     | `camelCase`                    |
| Classes            | `PascalCase`    | `PascalCase`     | `PascalCase`                   |
| Interfaces         | `IPascalCase`   | `PascalCase`     | `PascalCase` (no I prefix)     |
| Private fields     | `_camelCase`    | `_snake_case`    | `#field` or `_camelCase`       |
| Files              | `PascalCase.cs` | `snake_case.py`  | `camelCase.ts` / `kebab-case`  |

When planning file names, always use the correct convention for the target language.

---

## Angular 21 Planning Notes

When a plan involves Angular frontend:

- Default to **Standalone Components** (no NgModules unless existing codebase uses them)
- State management: **Angular Signals** for local state, consider NgRx only for global/shared state
- Forms: Reactive Forms for complex validation, Template-driven for simple inputs
- File naming: `kebab-case.component.ts`, `kebab-case.service.ts`
- Call **Woz** before finalizing any plan with UI components (as per Spock's standard workflow)

---

## Azure DevOps Pipeline Planning

When a feature requires pipeline changes, include in the plan:

- Stage affected: `CI` (build/test), `CD` (deploy), or both
- Environment: `dev`, `staging`, `prod`
- New variables or secrets needed (flag for manual creation in Azure DevOps)
- Approval gates if deploying to staging or prod

---

## Standard Risk Categories

Use these in the Risk Assessment table of every plan:

| Category              | Examples                                                  |
|-----------------------|-----------------------------------------------------------|
| Breaking Change       | API contract change, DB column rename/removal             |
| Data Migration        | Existing rows need transformation                         |
| External Dependency   | Third-party API, Telegram MTProto, Azure service          |
| Cross-Feature Impact  | Shared handler, shared table, shared abstraction modified |
| Performance           | N+1 query, missing index, large payload                   |
| Security              | New endpoint without auth, sensitive data exposure        |

---

## References

- [cqrs-without-mediatR.md](./references/cqrs-without-mediatR.md) — Full manual CQRS pattern with DI registration
- [architecture-selection-guide.md](./references/architecture-selection-guide.md) — Decision tree with trade-off analysis
