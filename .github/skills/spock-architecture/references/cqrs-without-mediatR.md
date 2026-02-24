# CQRS Without MediatR — Approved Pattern

Manual CQRS implementation for this project. MediatR is prohibited (commercial licence
from v12+). This pattern provides the same separation of commands and queries with
explicit, dependency-injected handlers and no hidden pipeline magic.

---

## Core Abstractions

These interfaces live in `Shared/Abstractions/` and are the only shared contract
across all features. **Do not add methods or generic constraints beyond what is here.**

```
ICommandHandler<TCommand, TResult>
  Task<TResult> HandleAsync(TCommand command, CancellationToken ct)

IQueryHandler<TQuery, TResult>
  Task<TResult> HandleAsync(TQuery query, CancellationToken ct)
```

---

## Vertical Slice Example — CreateOrder

### Files to plan

```
Features/Orders/CreateOrder/
  CreateOrderCommand.cs         // Input: CustomerId, Items
  CreateOrderHandler.cs         // Implements ICommandHandler<CreateOrderCommand, Result<Guid>>
  CreateOrderValidator.cs       // Business rule validation (manual)
  CreateOrderEndpoint.cs        // Minimal API — injects ICommandHandler directly
```

### Structural sketch (for Spock planning reference only)

**Command** — plain record, no behaviour:
```
record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items)
```

**Handler** — single responsibility, one public method:
```
class CreateOrderHandler : ICommandHandler<CreateOrderCommand, Result<Guid>>
  dependencies: AppDbContext, IOrderValidator
  HandleAsync → validate → create aggregate → save → return Result
```

**Endpoint** — thin slice, no business logic:
```
POST /api/orders
  injects: ICommandHandler<CreateOrderCommand, Result<Guid>>
  maps Result to 201 Created or 400 BadRequest
```

**DI Registration** — in `Program.cs` or a feature extension method:
```
services.AddScoped<ICommandHandler<CreateOrderCommand, Result<Guid>>, CreateOrderHandler>()
```

---

## Query Example — GetOrderById

```
Features/Orders/GetOrder/
  GetOrderQuery.cs              // Input: Guid OrderId
  GetOrderHandler.cs            // Implements IQueryHandler<GetOrderQuery, Result<OrderResponse>>
  OrderResponse.cs              // DTO — only fields needed by the caller
  GetOrderEndpoint.cs           // GET /api/orders/{id}
```

**Handler behaviour:**
```
GetOrderHandler : IQueryHandler<GetOrderQuery, Result<OrderResponse>>
  → db.Orders.AsNoTracking()
      .Where(o => o.Id == query.OrderId)
      .Select(o => new OrderResponse(...))   // project, never load full entity
      .FirstOrDefaultAsync()
  → return Result.NotFound() if null
  → return Result.Success(response)
```

---

## Result Pattern

```
Result<T>
  bool IsSuccess
  T? Value
  string? Error

  static Result<T> Success(T value)
  static Result<T> Failure(string error)
  static Result<T> NotFound(Guid id)    // optional — add if project uses it
```

Endpoint mapping convention:
```
IsSuccess  → 200 OK / 201 Created (with Location header for POST)
!IsSuccess, error is validation → 400 Bad Request
!IsSuccess, error is not-found  → 404 Not Found
```

---

## DI Registration Strategy

For small projects: register handlers one by one in `Program.cs`.

For larger projects, plan a **feature module extension method** per feature folder:
```
static class OrdersModule
  static IServiceCollection AddOrders(this IServiceCollection services)
    services.AddScoped<ICommandHandler<CreateOrderCommand, Result<Guid>>, CreateOrderHandler>()
    services.AddScoped<IQueryHandler<GetOrderQuery, Result<OrderResponse>>, GetOrderHandler>()
    return services
```

Then `Program.cs` calls: `builder.Services.AddOrders()`

This keeps `Program.cs` clean and makes features self-contained.

---

## What NOT to plan

- ❌ Pipeline behaviours (logging, validation, retry via MediatR pipeline)
- ❌ `INotification` / event bus through MediatR
- ❌ `ISender` or `IMediator` injection anywhere

**Alternatives to plan instead:**

| MediatR feature     | Approved alternative                                           |
|---------------------|----------------------------------------------------------------|
| Pipeline behaviour  | Decorator pattern on `ICommandHandler` or middleware           |
| Notifications       | Domain Events via EF Core interceptors or explicit service call|
| Fire-and-forget     | `IBackgroundTaskQueue` + hosted service                        |

---

## Validation Approach

Validation is the handler's responsibility (or a dedicated validator injected into the handler).
Do not plan a shared validation pipeline — keep it explicit per feature.

```
IOrderValidator
  Task<ValidationResult> ValidateAsync(CreateOrderCommand command, CancellationToken ct)

ValidationResult
  bool IsValid
  IReadOnlyList<string> Errors
```

If the project already uses FluentValidation, plan a validator class inheriting
`AbstractValidator<T>` and call `.Validate()` inside the handler. Do not add
FluentValidation to a project that does not already have it.
