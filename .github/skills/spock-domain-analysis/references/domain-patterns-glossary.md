# Domain Patterns Glossary

Canonical names for DDD patterns and CQRS concepts as used in this codebase.
Use this to ensure consistent naming when proposing new files and concepts in plans.

---

## CQRS Concepts

| Concept        | This codebase uses      | Never use           | Notes                                        |
|----------------|-------------------------|---------------------|----------------------------------------------|
| Write input    | `{Feature}Command`      | `{Feature}Request`  | Records, never classes                       |
| Write handler  | `{Feature}Handler`      | `{Feature}Service`  | Implements `ICommandHandler<TCmd, TResult>`  |
| Read input     | `{Feature}Query`        | `{Feature}Request`  | Records, never classes                       |
| Read handler   | `{Feature}QueryHandler` | `{Feature}Service`  | Implements `IQueryHandler<TQuery, TResult>`  |
| Read output    | `{Feature}Response`     | `{Feature}ViewModel`| DTO — only fields the client needs           |
| API endpoint   | `{Feature}Endpoint`     | `{Feature}Controller` | Minimal API static class                   |
| Validator      | `{Feature}Validator`    | `{Feature}Rules`    | Manual or inherits AbstractValidator<T>      |

---

## DDD Concepts

| Concept          | How it appears in C#                       | Location (VSA)                    | Location (Onion)      |
|------------------|--------------------------------------------|-----------------------------------|-----------------------|
| Entity           | Class with `Id` property (Guid)            | Shared/Domain/                    | Domain/Entities/      |
| Aggregate Root   | Entity with methods that enforce invariants| Shared/Domain/ or Feature folder  | Domain/Entities/      |
| Value Object     | `record` with no identity                  | Shared/Domain/ValueObjects/       | Domain/ValueObjects/  |
| Domain Event     | `record` implementing `IDomainEvent`       | Shared/Domain/Events/             | Domain/Events/        |
| Repository Iface | `I{Entity}Repository`                      | Shared/Abstractions/              | Domain/Interfaces/    |
| Repository Impl  | `{Entity}Repository`                       | Shared/Infrastructure/            | Infrastructure/       |
| Domain Service   | Stateless class for multi-aggregate logic  | Features/{Domain}/                | Domain/Services/      |

---

## Result Pattern Vocabulary

| Term               | Meaning                                                    |
|--------------------|------------------------------------------------------------|
| `Result.Success`   | Operation completed, value is available                    |
| `Result.Failure`   | Business rule violated — client error, 400 response        |
| `Result.NotFound`  | Entity not found — 404 response                           |
| `IsSuccess`        | Boolean — true for Success, false for Failure/NotFound     |
| `Value`            | The returned data — only valid when `IsSuccess` is true    |
| `Error`            | Human-readable error message — only valid when not success |

Never use exceptions to signal business failures. Exceptions are for unexpected
infrastructure errors only.

---

## File Naming Reference

### C# files

| File type                    | Naming pattern                       | Example                          |
|------------------------------|--------------------------------------|----------------------------------|
| Command record               | `{Feature}Command.cs`                | `CreateOrderCommand.cs`          |
| Command handler              | `{Feature}Handler.cs`                | `CreateOrderHandler.cs`          |
| Query record                 | `{Feature}Query.cs`                  | `GetOrderByIdQuery.cs`           |
| Query handler                | `{Feature}QueryHandler.cs`           | `GetOrderByIdQueryHandler.cs`    |
| Response DTO                 | `{Feature}Response.cs`               | `GetOrderByIdResponse.cs`        |
| Minimal API endpoint         | `{Feature}Endpoint.cs`               | `CreateOrderEndpoint.cs`         |
| Validator                    | `{Feature}Validator.cs`              | `CreateOrderValidator.cs`        |
| Entity                       | `{EntityName}.cs`                    | `Order.cs`                       |
| Value Object                 | `{ConceptName}.cs`                   | `Address.cs`, `Money.cs`         |
| Domain Event                 | `{EventName}Event.cs`                | `OrderCreatedEvent.cs`           |
| EF Configuration             | `{Entity}Configuration.cs`           | `OrderConfiguration.cs`          |
| Domain Module                | `{Domain}Module.cs`                  | `OrdersModule.cs`                |
| Test class                   | `{Subject}Tests.cs`                  | `CreateOrderHandlerTests.cs`     |

### TypeScript files

| File type          | Naming pattern                | Example                        |
|--------------------|-------------------------------|--------------------------------|
| Component          | `{name}.component.ts`         | `order-list.component.ts`      |
| Service            | `{name}.service.ts`           | `orders.service.ts`            |
| Model / DTO        | `{name}.model.ts`             | `order.model.ts`               |
| Guard              | `{name}.guard.ts`             | `auth.guard.ts`                |
| Pipe               | `{name}.pipe.ts`              | `currency-format.pipe.ts`      |
| Store (NgRx)       | `{name}.store.ts`             | `orders.store.ts`              |
| Spec               | `{name}.spec.ts`              | `orders.service.spec.ts`       |

### Python files

| File type          | Naming pattern                | Example                        |
|--------------------|-------------------------------|--------------------------------|
| Module             | `snake_case.py`               | `order_handler.py`             |
| Dataclass model    | `snake_case.py`               | `order_models.py`              |
| DSPy module        | `snake_case_module.py`        | `chain_of_density_module.py`   |
| Test file          | `test_{name}.py`              | `test_order_handler.py`        |

---

## Prohibited Patterns and Their Replacements

| Prohibited                          | Reason                              | Use instead                        |
|-------------------------------------|-------------------------------------|------------------------------------|
| `IMediator` / `ISender`             | MediatR is commercial from v12+     | Direct `ICommandHandler` injection |
| `IRequest<T>` / `IRequestHandler<T>`| MediatR types                       | `ICommandHandler<TCmd, TResult>`   |
| `AbstractValidator<T>` (new proj)   | FluentValidation — avoid new dep    | Manual validation or DataAnnotations |
| Service Locator                     | Hides dependencies                  | Constructor injection              |
| Static helper classes with state    | Not testable                        | Inject as interface                |
| God class / God handler             | Violates SRP                        | Split into focused handlers        |
| Nested functions (Python)           | Hard to test, hidden scope          | Module-level functions             |
| `from x import *`                   | Pollutes namespace, unclear deps    | Explicit imports                   |
| `var` (TypeScript)                  | Avoid function scope                | `const` (default) or `let`        |
| `any` (TypeScript)                  | Loses type safety                   | Explicit type or `unknown`         |
