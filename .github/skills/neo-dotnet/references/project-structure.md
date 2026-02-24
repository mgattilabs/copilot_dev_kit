# Project Structure

## Solution Layout

The solution follows Clean Architecture layer separation with one project per layer.
Each project has a clear dependency direction: Presentation → Infrastructure → Application → Domain.

```
src/
├── MyApp.Domain/                          # Innermost layer — ZERO dependencies
│   ├── Common/                            # Shared domain building blocks
│   │   ├── AggregateRoot.cs
│   │   ├── IDomainEvent.cs
│   │   ├── IUnitOfWork.cs
│   │   └── DomainException.cs
│   ├── Orders/                            # Bounded context: Orders
│   │   ├── Order.cs                       # Aggregate root
│   │   ├── OrderItem.cs                   # Entity inside aggregate
│   │   ├── OrderId.cs                     # Strongly-typed ID
│   │   ├── OrderStatus.cs                 # Enum
│   │   ├── IOrderRepository.cs            # Repository interface (contract)
│   │   ├── IOrderPricingService.cs        # Domain service interface
│   │   └── Events/
│   │       ├── OrderCreatedEvent.cs
│   │       └── OrderConfirmedEvent.cs
│   ├── Customers/                         # Bounded context: Customers
│   │   ├── Customer.cs
│   │   ├── CustomerId.cs
│   │   ├── Email.cs                       # Value object
│   │   └── ICustomerRepository.cs
│   └── SharedKernel/                      # Value objects shared across contexts
│       ├── Money.cs
│       ├── Address.cs
│       └── Quantity.cs
│
├── MyApp.Application/                     # Use cases — depends only on Domain
│   ├── Common/                            # Shared application infrastructure
│   │   ├── IRepository.cs                 # Generic repository interface
│   │   └── PagedResult.cs
│   ├── Orders/                            # Matches Domain bounded context
│   │   ├── Commands/
│   │   │   ├── CreateOrderCommand.cs      # Command record
│   │   │   ├── CreateOrderHandler.cs      # Handler (no MediatR)
│   │   │   ├── CancelOrderCommand.cs
│   │   │   └── CancelOrderHandler.cs
│   │   ├── Queries/
│   │   │   ├── GetOrderByIdHandler.cs
│   │   │   └── ListOrdersHandler.cs
│   │   ├── Dtos/
│   │   │   ├── OrderDetailDto.cs
│   │   │   ├── OrderSummaryDto.cs
│   │   │   └── OrderItemDto.cs
│   │   └── EventHandlers/
│   │       └── OrderConfirmedEventHandler.cs
│   ├── Customers/
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── Dtos/
│   └── DependencyInjection.cs             # AddApplicationServices()
│
├── MyApp.Infrastructure/                  # Implements interfaces — depends on Domain + Application
│   ├── Persistence/
│   │   ├── AppDbContext.cs                # DbContext + IUnitOfWork
│   │   ├── Configurations/               # EF Core entity configurations
│   │   │   ├── OrderConfiguration.cs
│   │   │   ├── OrderItemConfiguration.cs
│   │   │   └── CustomerConfiguration.cs
│   │   ├── Repositories/
│   │   │   ├── RepositoryBase.cs          # Generic implementation
│   │   │   ├── OrderRepository.cs
│   │   │   └── CustomerRepository.cs
│   │   └── Migrations/                    # EF Core migrations
│   ├── Services/
│   │   ├── OrderPricingService.cs         # Domain service implementation
│   │   └── EmailService.cs               # External service integration
│   └── DependencyInjection.cs             # AddInfrastructureServices()
│
└── MyApp.Api/                             # Presentation — outermost layer
    ├── Program.cs                         # Composition root
    ├── appsettings.json
    ├── Endpoints/                         # Minimal API route groups
    │   ├── OrderEndpoints.cs
    │   └── CustomerEndpoints.cs
    ├── Middleware/
    │   └── GlobalExceptionHandler.cs
    ├── Requests/                          # Input DTOs with Data Annotations
    │   ├── CreateOrderRequest.cs
    │   └── CreateCustomerRequest.cs
    └── Properties/
        └── launchSettings.json

tests/
├── MyApp.Domain.Tests/                    # Unit tests — domain logic
│   ├── Orders/
│   │   ├── OrderTests.cs
│   │   └── MoneyTests.cs
│   └── Customers/
│       └── EmailTests.cs
├── MyApp.Application.Tests/               # Unit tests — use cases (mocked repos)
│   └── Orders/
│       ├── CreateOrderHandlerTests.cs
│       └── GetOrderByIdHandlerTests.cs
└── MyApp.Api.Tests/                       # Integration tests — API endpoints
    ├── OrderEndpointsTests.cs
    └── Infrastructure/
        └── TestWebApplicationFactory.cs
```

## Project References

The `.csproj` references enforce the dependency rule at compile time.
Any violation is a build error.

**MyApp.Domain.csproj** references nothing — it is the core.

**MyApp.Application.csproj** references only `MyApp.Domain`.

**MyApp.Infrastructure.csproj** references `MyApp.Domain` and `MyApp.Application`.
It also has NuGet references to `Microsoft.EntityFrameworkCore.SqlServer` and other
infrastructure packages.

**MyApp.Api.csproj** references `MyApp.Application` and `MyApp.Infrastructure`
(for DI registration). It has NuGet references to ASP.NET Core packages.

## Target Framework

All projects target .NET 10:

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

## Naming Conventions

**Projects** use the format `{SolutionName}.{Layer}`:
`MyApp.Domain`, `MyApp.Application`, `MyApp.Infrastructure`, `MyApp.Api`.

**Folders inside projects** match bounded contexts from the domain:
`Orders/`, `Customers/`, `Products/`.

**Files** are named after the single class/record they contain. One type per file.

**Handlers** are named `{Action}{Entity}Handler.cs`:
`CreateOrderHandler.cs`, `GetOrderByIdHandler.cs`.

**Commands/Queries** are named `{Action}{Entity}Command.cs` or `{Action}{Entity}Query.cs`
(queries can omit the suffix when the handler name is clear enough).

**DTOs** are named `{Entity}{Purpose}Dto.cs`:
`OrderDetailDto.cs`, `OrderSummaryDto.cs`.

**Requests** (API input models) are named `{Action}{Entity}Request.cs`:
`CreateOrderRequest.cs`.

**Configurations** are named `{Entity}Configuration.cs`:
`OrderConfiguration.cs`.

**Repositories** are named `{Entity}Repository.cs` with interface `I{Entity}Repository.cs`.

## File Organization Rules

Each bounded context is self-contained within its folder. You should be able to
understand a bounded context by reading only its folder, without jumping to
unrelated parts of the solution.

Inside Application, Commands and Queries are separated into subfolders. This makes
it immediately clear which operations modify state and which are read-only.

The `Common/` folder in each project holds shared building blocks that are used
across multiple bounded contexts — base classes, marker interfaces, generic utilities.

Tests mirror the source structure. `MyApp.Domain.Tests/Orders/OrderTests.cs` tests
the logic in `MyApp.Domain/Orders/Order.cs`.
