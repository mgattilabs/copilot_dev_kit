# Architecture Selection Guide

Decision framework for Spock when choosing between Vertical Slice Architecture (VSA)
and Onion / Clean Architecture. Use this during Phase 1 (Interview) to identify which
architecture the project uses, and during Phase 2 (Plan) to justify the approach.

---

## Decision Tree

```
Is the project already started?
│
├── YES → Identify existing architecture from folder structure
│         ├── Features/ at root → Vertical Slice → stay with VSA
│         ├── Domain/ Application/ Infrastructure/ → Onion → stay with Onion
│         └── Mixed or unclear → flag in plan as ⚠️ ASSUMPTION, ask in interview
│
└── NO (greenfield) → Apply selection criteria below
```

### Greenfield Selection Criteria

Ask these during Phase 1 (Interview):

| Question                                              | VSA signal        | Onion signal             |
|-------------------------------------------------------|-------------------|--------------------------|
| How complex is the business domain?                   | Low–Medium        | High (rich domain model) |
| Will multiple delivery mechanisms exist? (API + CLI)  | No                | Yes                      |
| Will the domain be tested independently of the DB?    | Not a priority    | Critical requirement     |
| How many developers will work on this?                | 1–3               | 4+, separate domain team |
| Expected project lifespan?                            | < 3 years         | 3+ years                 |
| Is speed of initial delivery the top priority?        | Yes               | No                       |

**Tie-break rule:** When in doubt, default to **Vertical Slice**. It is simpler to
evolve a VSA project toward Onion than to simplify an over-engineered Onion project.

---

## Vertical Slice Architecture

### Characteristics

- Each feature is self-contained: command/query/handler/endpoint in one folder
- No shared application layer — features share only infrastructure (DbContext, abstractions)
- Adding a feature = adding a folder; deleting a feature = deleting a folder
- Dependencies between features are explicit and visible

### Folder template

```
src/
  Features/
    {Domain}/
      {Feature}/
        {Feature}Command.cs
        {Feature}Handler.cs
        {Feature}Validator.cs       // optional
        {Feature}Endpoint.cs
      {Feature}/
        {Feature}Query.cs
        {Feature}QueryHandler.cs
        {Feature}Response.cs
        {Feature}Endpoint.cs
  Shared/
    Abstractions/
      ICommandHandler.cs
      IQueryHandler.cs
      Result.cs
    Infrastructure/
      AppDbContext.cs
      AppDbContextFactory.cs        // for EF migrations
  WebApi/
    Program.cs
    Modules/
      {Domain}Module.cs             // DI + route registration per domain
tests/
  Features/
    {Domain}/
      {Feature}HandlerTests.cs
      {Feature}ValidatorTests.cs
```

### When Spock plans a new feature in VSA

1. Identify the domain folder (e.g. `Orders`, `Customers`)
2. Create a new feature subfolder
3. List all files to CREATE with their purpose
4. Check if `{Domain}Module.cs` needs updating (DI registration + route mapping)
5. Check if `AppDbContext` needs a new `DbSet<>` or configuration

---

## Onion / Clean Architecture

### Characteristics

- Strict layer isolation: Domain → Application → Infrastructure → WebApi
- Domain has zero dependencies (no EF Core, no external packages)
- Application layer defines interfaces; Infrastructure implements them
- Long compile-time boundary enforcement via project references

### Project structure

```
src/
  {ProjectName}.Domain/
    Entities/
    ValueObjects/
    Events/
    Interfaces/         // repository interfaces defined here
    Exceptions/
  {ProjectName}.Application/
    UseCases/
      {Feature}/
        {Feature}Command.cs
        {Feature}Handler.cs         // implements ICommandHandler (from Shared)
        {Feature}Query.cs
        {Feature}QueryHandler.cs
    Abstractions/
      ICommandHandler.cs
      IQueryHandler.cs
      Result.cs
    Interfaces/                     // IUnitOfWork, IEmailService, etc.
  {ProjectName}.Infrastructure/
    Persistence/
      AppDbContext.cs
      Configurations/               // IEntityTypeConfiguration<T> per entity
      Repositories/                 // implement Domain interfaces
    ExternalServices/
  {ProjectName}.WebApi/
    Endpoints/
      {Domain}/
        {Feature}Endpoint.cs
    Program.cs
tests/
  {ProjectName}.Domain.Tests/
  {ProjectName}.Application.Tests/
  {ProjectName}.Infrastructure.Tests/
```

### Dependency rule (enforced via .csproj references)

```
WebApi      → Infrastructure, Application
Infrastructure → Application, Domain
Application → Domain
Domain      → (nothing)
```

Any plan that adds a reference pointing outward is a **blocker**. Flag it with ⚠️.

### When Spock plans a new feature in Onion

1. Identify which layers are affected (usually Application + Infrastructure + WebApi)
2. Check if a new Domain entity is needed → plan it in `Domain/Entities/`
3. Define the interface in `Application/Interfaces/` if a new external capability is needed
4. Plan the handler in `Application/UseCases/{Feature}/`
5. Plan the infrastructure implementation in `Infrastructure/`
6. Plan the endpoint in `WebApi/Endpoints/{Domain}/`
7. Check if a new migration is needed

---

## Common Planning Mistakes to Avoid

| Mistake                                          | Correct approach                                          |
|--------------------------------------------------|-----------------------------------------------------------|
| Putting business logic in endpoints              | Business logic belongs in handlers (VSA) or domain (Onion)|
| Sharing a handler between two features           | Duplicate if needed; coupling features is worse           |
| Adding EF Core to Domain project in Onion        | EF belongs in Infrastructure                              |
| Skipping DI registration in the plan             | Always include the DI step — Neo misses it otherwise      |
| Planning MediatR IRequest/IRequestHandler        | Use manual ICommandHandler / IQueryHandler                |
| Planning FluentValidation for a new project      | Use DataAnnotations or manual validation instead          |
| Putting Domain Events in Application layer       | Domain Events are defined in Domain layer                 |

---

## Architecture-Agnostic Rules (Always Apply)

These apply regardless of VSA or Onion:

- **One handler = one responsibility.** If a handler touches two unrelated aggregates, split it.
- **Read models are separate from write models.** Never reuse a command DTO as a query response.
- **Async all the way.** Every handler method is `async Task<T>`. No `.Result` or `.Wait()`.
- **Result pattern everywhere.** No exceptions for business logic — use `Result<T>`.
- **Schema changes = migration step.** Any DB column, table, or index change needs an explicit
  migration step in the plan, listed as a separate phase item.
