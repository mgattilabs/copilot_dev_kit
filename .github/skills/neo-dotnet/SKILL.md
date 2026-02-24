---
name: neo-dotnet
description: >
  .NET and C# expertise for Neo. Activate when writing C# code, creating .NET
  endpoints, implementing CQRS with MediatR, Entity Framework Core, Minimal API,
  Vertical Slice Architecture, Clean/Onion Architecture, FluentValidation, xUnit
  tests, NuGet packages, ASP.NET Core middleware, or any .cs / .csproj file.
---

# Neo .NET Expertise

## Naming Conventions (Company Standard)

```csharp
public class UserService { }           // Classes: PascalCase
public interface IUserRepository { }   // Interfaces: IPascalCase
public void GetUserById(int id) { }    // Methods: PascalCase
string userName = "Mario";             // Local vars: camelCase
public string FirstName { get; set; } // Properties: PascalCase
private readonly ILogger _logger;      // Private fields: _camelCase
public const int MaxRetryCount = 3;   // Constants: PascalCase
public enum UserStatus { Active, Inactive, Pending }
```

## Architecture Decision

Use **Vertical Slice Architecture** (default) unless the project explicitly uses Clean/Onion.

### Vertical Slice — Feature Folder Structure
```
Features/
  Orders/
    CreateOrder/
      CreateOrderCommand.cs      // MediatR Command
      CreateOrderHandler.cs      // Handler
      CreateOrderValidator.cs    // FluentValidation
      CreateOrderEndpoint.cs     // Minimal API endpoint
    GetOrder/
      GetOrderQuery.cs
      GetOrderHandler.cs
      GetOrderResponse.cs
```

### Clean/Onion — Layer Structure
```
Domain/          // Entities, Value Objects, Domain Events — ZERO dependencies
Application/     // Use Cases, Commands, Queries, Interfaces
Infrastructure/  // EF Core, External APIs, implementations of Application interfaces
WebApi/          // Controllers or Minimal API, DI configuration
```
**Rule**: Dependencies ALWAYS point inward → Domain has zero outward dependencies.

## CQRS with MediatR

```csharp
// Command
public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items) 
    : IRequest<Result<Guid>>;

// Handler
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    private readonly AppDbContext _db;
    private readonly IPublisher _publisher;

    public CreateOrderHandler(AppDbContext db, IPublisher publisher)
    {
        _db = db;
        _publisher = publisher;
    }

    public async Task<Result<Guid>> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Items);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        await _publisher.Publish(new OrderCreatedEvent(order.Id), ct);
        return Result.Success(order.Id);
    }
}
```

## Minimal API Endpoint Pattern

```csharp
public static class CreateOrderEndpoint
{
    public static void MapCreateOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (
            CreateOrderCommand command,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(command, ct);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("CreateOrder")
        .WithTags("Orders")
        .Produces<Guid>(201)
        .ProducesProblem(400)
        .RequireAuthorization();
    }
}
```

## FluentValidation

```csharp
public class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("Customer ID is required");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item")
            .ForEach(item => item
                .ChildRules(i =>
                {
                    i.RuleFor(x => x.Quantity).GreaterThan(0);
                    i.RuleFor(x => x.ProductId).NotEmpty();
                }));
    }
}
```

## Entity Framework Core

```csharp
// ✅ Efficient query — AsNoTracking for reads
var orders = await _db.Orders
    .AsNoTracking()
    .Include(o => o.Items)
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Active)
    .OrderByDescending(o => o.CreatedAt)
    .ToListAsync(ct);

// ✅ Projection — select only what you need
var orderSummaries = await _db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status, o.CreatedAt))
    .ToListAsync(ct);

// ❌ Never use .Result or .Wait() — always await
// ❌ Never load full entity for read-only operations
```

## Clean Code Rules (Company Specific)

1. **Single Responsibility**: one method → one task
2. **Early Return**: handle edge cases first, keep happy path clean
3. **No Magic Numbers**: use named constants
4. **Expressive Conditions**: extract complex conditions to named variables
5. **Comments only for WHY**: never comment what the code does, only why

```csharp
// ❌ Magic number
if (elapsedSeconds > 86400) ExpireSession();

// ✅ Named constant
private const int SecondsInADay = 86400;
if (elapsedSeconds > SecondsInADay) ExpireSession();

// ❌ Nested hell
public decimal CalculateDiscount(Customer customer, Order order)
{
    decimal discount = 0;
    if (customer != null) { if (order != null) { if (order.Total > 100) { ... } } }
    return discount;
}

// ✅ Early return
public decimal CalculateDiscount(Customer customer, Order order)
{
    if (customer == null || order == null) return 0;
    if (order.Total <= 100) return 0;
    return customer.IsVip ? 0.2m : 0.1m;
}
```

## xUnit Testing Pattern

```csharp
public class CreateOrderHandlerTests
{
    private readonly AppDbContext _db;
    private readonly CreateOrderHandler _handler;

    public CreateOrderHandlerTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;
        _db = new AppDbContext(options);
        _handler = new CreateOrderHandler(_db, Substitute.For<IPublisher>());
    }

    [Fact]
    public async Task Handle_ValidCommand_CreatesOrderAndReturnsId()
    {
        // Arrange
        var command = new CreateOrderCommand(Guid.NewGuid(), 
            [new OrderItemDto(Guid.NewGuid(), 2)]);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        var order = await _db.Orders.FindAsync(result.Value);
        order.Should().NotBeNull();
    }
}
```

## NuGet Packages (Standard Stack)

```xml
<!-- Core -->
<PackageReference Include="MediatR" Version="12.*" />
<PackageReference Include="FluentValidation.AspNetCore" Version="11.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.*" />

<!-- Testing -->
<PackageReference Include="xunit" Version="2.*" />
<PackageReference Include="FluentAssertions" Version="6.*" />
<PackageReference Include="NSubstitute" Version="5.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="9.*" />
```

## Error Handling Pattern

```csharp
// Use Result pattern — never throw for business logic
public record Result<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }

    public static Result<T> Success(T value) => new() { IsSuccess = true, Value = value };
    public static Result<T> Failure(string error) => new() { IsSuccess = false, Error = error };
}

// Global exception middleware for unexpected errors
app.UseExceptionHandler(errApp =>
{
    errApp.Run(async ctx =>
    {
        ctx.Response.StatusCode = 500;
        await ctx.Response.WriteAsJsonAsync(new { error = "An unexpected error occurred" });
    });
});
```

## References
- See [cqrs-patterns.md](./references/cqrs-patterns.md) for advanced CQRS examples
- See [minimal-api-templates.md](./references/minimal-api-templates.md) for full endpoint templates
