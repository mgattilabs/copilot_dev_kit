# .NET / C# Skill — Neo Backend

> **Trigger**: `*.sln` o `*.csproj` trovato nella root del progetto
> **Stack**: C# 14 / .NET (ultima versione LTS)
> **Questa skill estende le regole generiche di Neo Backend con convenzioni .NET-specifiche.**

---

## Stack Obbligatorio

| Categoria       | Tecnologia                                     | Note                                    |
|-----------------|------------------------------------------------|-----------------------------------------|
| Linguaggio      | C# 14 / .NET (ultima versione LTS)             | Nullable enabled, ImplicitUsings enabled|
| API             | ASP.NET Core Minimal API                       | MAI Controller-based in progetti nuovi  |
| ORM             | Entity Framework Core (ultima versione)        | SQL Server o SQLite in dev              |
| CQRS            | **Manuale — ICommandHandler / IQueryHandler**  | ⚠️ MediatR è VIETATO                    |
| Validazione     | DataAnnotations o manuale                      | FluentValidation solo se già presente   |
| Error handling  | Result Pattern                                 | MAI eccezioni per business logic        |
| Testing         | xUnit + NSubstitute + FluentAssertions         | Un test per ogni handler                |
| DI              | Constructor injection con `IServiceCollection` | MAI Service Locator                     |

---

## ⚠️ MediatR è VIETATO

MediatR è diventato commerciale dalla v12+. Non pianificare, non usare, non
suggerire `IRequest<T>`, `IRequestHandler<T>`, `IMediator`, `ISender` o qualsiasi
tipo di MediatR. Il pattern approvato è il CQRS manuale documentato in questa skill.

---

## Architecture Invariants (NON MODIFICARE SENZA RICHIESTA ESPLICITA)

Queste invarianti valgono per **tutti** i progetti .NET. Non si cambiano, non si semplificano,
non si accorpano. Qualsiasi piano che le viola è un errore.

### 1. Clean Architecture — Quattro progetti, sempre

Ogni progetto .NET usa la struttura Clean Architecture con esattamente quattro progetti:

```
sources/
├── [NomeProgetto].Api/              // Endpoint, middleware, composition root
├── [NomeProgetto].Application/      // Use cases, handler, abstrazioni
├── [NomeProgetto].Domain/           // Entità, value objects, eventi, interfacce
└── [NomeProgetto].Infrastructure/   // Persistenza, servizi esterni
```

- `[NomeProgetto]` è il nome della soluzione (es. `ViaFrancigena`, `ContoCorso`, ecc.)
- La cartella radice è `sources/`, non `src/`
- **Mai accorpare progetti** — anche per feature piccole, i quattro progetti esistono sempre
- **Mai aggiungere un quinto progetto** senza richiesta esplicita

### 2. Split per App Context — Api e Application

Il layer **Api** e il layer **Application** sono organizzati per **contesto applicativo**:

```
sources/[NomeProgetto].Api/
├── Endpoints/
│   ├── Cms/              // Backoffice / content management
│   ├── Website/          // Sito pubblico
│   ├── PWA/              // Progressive Web App
│   ├── Planner/          // Pianificatore
│   └── Identity/         // Auth, utenti, ruoli
└── ...

sources/[NomeProgetto].Application/
├── UseCases/
│   ├── Cms/
│   │   └── {Feature}/
│   ├── Website/
│   │   └── {Feature}/
│   ├── PWA/
│   │   └── {Feature}/
│   ├── Planner/
│   │   └── {Feature}/
│   └── Identity/
│       └── {Feature}/
└── ...
```

- I contesti possono variare tra progetti, ma la struttura per-contesto è obbligatoria
- Un endpoint appartiene sempre a un solo contesto
- Un handler in Application appartiene allo stesso contesto del suo endpoint

### 3. Split per Feature Interna — Domain e Infrastructure

Il layer **Domain** e il layer **Infrastructure** sono organizzati per **feature di dominio**
(non per contesto applicativo):

```
sources/[NomeProgetto].Domain/
├── Entities/
│   ├── Routes/           // Feature: percorsi
│   ├── Stages/           // Feature: tappe
│   ├── Users/            // Feature: utenti
│   └── ...
├── ValueObjects/
├── Events/
└── Interfaces/

sources/[NomeProgetto].Infrastructure/
├── Persistence/
│   └── Database/
│       ├── AppDbContext.cs
│       └── Configurations/          // ← un file per aggregate/entity
│           ├── RouteConfiguration.cs
│           ├── StageConfiguration.cs
│           ├── UserConfiguration.cs
│           └── ...
├── Repositories/
└── Services/
```

### 4. EF Core Configuration — Granularità obbligatoria

- **Un file di configurazione per aggregate/entity** in `sources/[NomeProgetto].Infrastructure/Persistence/Database/Configurations/`
- Il path è esattamente questo — non `Persistence/Configurations/`, non `Data/Configurations/`
- Mai configurazioni inline nel `DbContext` — sempre `IEntityTypeConfiguration<T>` in file separato

---

## Struttura Cartelle — Vertical Slice (dentro Clean Architecture)

> Le Architecture Invariants impongono **sempre** i 4 progetti Clean Architecture.
> Il Vertical Slice è il pattern organizzativo **interno** al layer Application:
> ogni feature è una cartella autocontenuta con command, handler, query, response.

```
sources/[NomeProgetto].Application/
└── UseCases/
    └── {AppContext}/                    // Cms, Website, PWA, Planner, Identity
        └── {Feature}/
            ├── {Feature}Command.cs      // Record input per write operations
            ├── {Feature}Handler.cs      // Implements ICommandHandler<TCmd, TResult>
            ├── {Feature}Validator.cs    // Validazione manuale (opzionale)
            ├── {Feature}Query.cs        // Record input per read operations
            ├── {Feature}QueryHandler.cs // Implements IQueryHandler<TQuery, TResult>
            └── {Feature}Response.cs     // DTO — solo campi necessari al client
```

Gli endpoint corrispondenti vivono nel layer Api, nello stesso contesto applicativo:

```
sources/[NomeProgetto].Api/
└── Endpoints/
    └── {AppContext}/
        └── {Feature}Endpoint.cs         // Minimal API — inietta handler direttamente
```

---

## Struttura Cartelle — Clean / Onion

> ⚠️ Questa è l'architettura obbligatoria. Vedi "Architecture Invariants" sopra per i vincoli non negoziabili.

```
sources/
├── [NomeProgetto].Domain/            // ZERO dipendenze esterne
│   ├── Entities/
│   │   └── {Feature}/                // Split per feature di dominio
│   ├── ValueObjects/
│   ├── Events/
│   └── Interfaces/                   // Repository interfaces definite qui
├── [NomeProgetto].Application/       // Dipende solo da Domain
│   ├── UseCases/
│   │   └── {AppContext}/             // Split per contesto applicativo (Cms, Website, PWA, ...)
│   │       └── {Feature}/
│   │           ├── {Feature}Command.cs
│   │           ├── {Feature}Handler.cs
│   │           ├── {Feature}Query.cs
│   │           ├── {Feature}QueryHandler.cs
│   │           └── {Feature}Response.cs
│   └── Abstractions/
│       ├── ICommandHandler.cs
│       ├── IQueryHandler.cs
│       └── Result.cs
├── [NomeProgetto].Infrastructure/    // Dipende da Domain + Application
│   ├── Persistence/
│   │   └── Database/
│   │       ├── AppDbContext.cs
│   │       └── Configurations/       // Un file per aggregate/entity
│   ├── Repositories/
│   └── Services/
└── [NomeProgetto].Api/               // Layer più esterno
    ├── Program.cs
    ├── Endpoints/
    │   └── {AppContext}/             // Split per contesto applicativo
    └── Middleware/
```

**Dipendenze**: `Api → Infrastructure → Application → Domain`. Domain non referenzia nulla.

---

## CQRS Manuale — Template Obbligatori

### Le due interfacce base (in `[NomeProgetto].Application/Abstractions/`)

```csharp
// Tutte le write operations
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct);
}

// Tutte le read operations
public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct);
}
```

### Command + Handler

```csharp
// Command — record, nessun comportamento
public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items);

// Handler — singola responsabilità, un solo metodo pubblico
public sealed class CreateOrderHandler(
    AppDbContext db,
    IOrderValidator validator)
    : ICommandHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> HandleAsync(
        CreateOrderCommand command, CancellationToken ct)
    {
        var validation = await validator.ValidateAsync(command, ct);
        if (!validation.IsValid)
            return Result<Guid>.Failure(string.Join(", ", validation.Errors));

        var order = Order.Create(command.CustomerId, command.Items);
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        return Result<Guid>.Success(order.Id);
    }
}
```

### Query + Handler

```csharp
// Query — record, nessun comportamento
public record GetOrderByIdQuery(Guid OrderId);

// Handler — legge con AsNoTracking + projection, mai entity completa
public sealed class GetOrderByIdQueryHandler(AppDbContext db)
    : IQueryHandler<GetOrderByIdQuery, Result<GetOrderByIdResponse>>
{
    public async Task<Result<GetOrderByIdResponse>> HandleAsync(
        GetOrderByIdQuery query, CancellationToken ct)
    {
        var response = await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new GetOrderByIdResponse(
                o.Id, o.CustomerId, o.TotalAmount, o.Status, o.CreatedAt))
            .FirstOrDefaultAsync(ct);

        return response is null
            ? Result<GetOrderByIdResponse>.NotFound(query.OrderId)
            : Result<GetOrderByIdResponse>.Success(response);
    }
}
```

### Minimal API Endpoint

```csharp
// Endpoint — thin slice, zero business logic
public static class CreateOrderEndpoint
{
    public static void MapCreateOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (
            CreateOrderCommand command,
            ICommandHandler<CreateOrderCommand, Result<Guid>> handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(command, ct);

            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("CreateOrder")
        .WithTags("Orders")
        .Produces<Guid>(201)
        .ProducesProblem(400)
        .RequireAuthorization();   // ← SEMPRE — nessun endpoint anonimo senza giustificazione
    }
}
```

### DI Registration — Feature Module

```csharp
// {Domain}Module.cs — ogni feature si registra da sola
public static class OrdersModule
{
    public static IServiceCollection AddOrders(this IServiceCollection services)
    {
        services.AddScoped<ICommandHandler<CreateOrderCommand, Result<Guid>>,
            CreateOrderHandler>();
        services.AddScoped<IQueryHandler<GetOrderByIdQuery, Result<GetOrderByIdResponse>>,
            GetOrderByIdQueryHandler>();
        return services;
    }

    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        app.MapCreateOrder();
        app.MapGetOrderById();
        return app;
    }
}

// Program.cs — composition root pulito
builder.Services.AddOrders();
app.MapOrders();
```

---

## Result Pattern — Template Obbligatorio

```csharp
// In [NomeProgetto].Application/Abstractions/Result.cs
public sealed record Result<T>
{
    public bool IsSuccess { get; private init; }
    public T? Value { get; private init; }
    public string? Error { get; private init; }

    private Result() { }

    public static Result<T> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error) =>
        new() { IsSuccess = false, Error = error };

    public static Result<T> NotFound(Guid id) =>
        new() { IsSuccess = false, Error = $"Entity with id '{id}' was not found." };
}

// Mapping Result → HTTP nei Minimal API Endpoints:
// IsSuccess              → 200 OK o 201 Created
// Failure (business)     → 400 BadRequest
// NotFound               → 404 NotFound
// Infrastructure failure → 500 (gestito dal GlobalExceptionHandler)
```

---

## Domain Model — Template DDD in C#

### Aggregate Root

```csharp
// Base class per tutti gli aggregate root
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent @event) => _domainEvents.Add(@event);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Aggregate — costruttore privato, factory method, setter privati
public sealed class Order : AggregateRoot
{
    public OrderId Id { get; private init; }
    public CustomerId CustomerId { get; private init; }
    public OrderStatus Status { get; private set; }

    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { }   // EF Core richiede il costruttore senza parametri

    public static Order Create(CustomerId customerId, IEnumerable<OrderItem> items)
    {
        var itemList = items.ToList();
        if (itemList.Count == 0)
            throw new DomainRuleViolationException("L'ordine deve contenere almeno un articolo.");

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
        };

        foreach (var item in itemList)
            order._items.Add(item);

        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleViolationException(
                $"Impossibile confermare un ordine in stato '{Status}'.");

        Status = OrderStatus.Confirmed;
        RaiseDomainEvent(new OrderConfirmedEvent(Id));
    }
}
```

### Strongly-Typed IDs

```csharp
// Ogni entity usa un ID tipizzato — mai Guid raw nel dominio
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value)
    {
        if (value == Guid.Empty)
            throw new DomainException("OrderId non può essere vuoto.");
        return new OrderId(value);
    }
    public override string ToString() => Value.ToString();
}
```

### Value Objects

```csharp
// Record immutabili con factory method e validazione
public sealed record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Money Create(decimal amount, string currency)
    {
        if (amount < 0)
            throw new DomainException("Il valore non può essere negativo.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("La valuta deve essere un codice ISO a 3 lettere.");
        return new Money(amount, currency.ToUpperInvariant());
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Impossibile sommare {Currency} e {other.Currency}.");
        return Create(Amount + other.Amount, Currency);
    }
}
```

---

## EF Core — Regole Obbligatorie

Queste regole si applicano a ogni singola query. Non esistono eccezioni.

Le query di lettura usano sempre `AsNoTracking()` e proiettano su DTO con `Select()`.
Non si carica mai un'entity completa per operazioni read-only.
Le query di scrittura usano le entity tracked attraverso il repository o direttamente
dal DbContext.

```csharp
// ✅ Read — AsNoTracking + projection
var summaries = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto(o.Id, o.Status, o.CreatedAt))
    .ToListAsync(ct);

// ✅ Single entity read — FindAsync per primary key
var order = await db.Orders.FindAsync([id], ct);

// ✅ Write — entity tracked, SaveChangesAsync con CancellationToken
db.Orders.Add(order);
await db.SaveChangesAsync(ct);

// ❌ MAI .Result o .Wait()
// ❌ MAI SELECT * (caricare entità complete per letture)
// ❌ MAI lazy loading — usare Include() esplicito o projection
// ❌ MAI IQueryable esposto fuori dall'Infrastructure layer
```

### EF Configuration — un file per entity

```csharp
// sources/[NomeProgetto].Infrastructure/Persistence/Database/Configurations/OrderConfiguration.cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        // Strongly-typed ID
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => OrderId.From(value));

        // Value Object come owned type
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount).HasColumnName("TotalAmount").HasPrecision(18, 2);
            money.Property(m => m.Currency).HasColumnName("Currency").HasMaxLength(3);
        });

        // Enum come stringa — mai come int
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        // Indici per colonne frequentemente filtrate
        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.CreatedAt);
    }
}
```

### EF Migrations — MAI creare, solo suggerire

Neo **non crea mai migrazioni EF Core** durante l'implementazione. Le migrazioni sono
un'operazione manuale che il developer esegue dopo aver verificato il codice.

Al termine dell'implementazione di una feature che modifica lo schema (nuove entity,
nuove proprietà, nuove configuration), Neo aggiunge un blocco nel riepilogo finale:

```
📦 MIGRAZIONE NECESSARIA
Le seguenti modifiche allo schema richiedono una migrazione EF Core:
- [elenco delle entity/configuration create o modificate]

Comando suggerito:
  dotnet ef migrations add [NomeDescrittivo] --project sources/[NomeProgetto].Infrastructure --startup-project sources/[NomeProgetto].Api

Dopo la migrazione, verificare il file generato prima di applicarlo.
```

**Regole:**
- ❌ MAI eseguire `dotnet ef migrations add` — solo suggerire il comando
- ❌ MAI eseguire `dotnet ef database update` — solo suggerire il comando
- ✅ Sempre indicare quali entity/configuration hanno impatto sullo schema
- ✅ Sempre proporre un nome descrittivo per la migrazione (es. `AddOrderEntity`, `AddStageStartDate`)

---

## Object Calisthenics — Esempi C#

```csharp
// ✅ Un livello di indentazione — estratto il body del loop
public void ProcessOrders(IEnumerable<Order> orders)
{
    foreach (var order in orders)
        ProcessSingleOrder(order);
}

// ✅ No else — guard clause + early return
public Result<Guid> ConfirmOrder(Order order)
{
    if (order is null) return Result<Guid>.Failure("Ordine non trovato.");
    if (order.Status != OrderStatus.Draft)
        return Result<Guid>.Failure("Solo gli ordini in bozza possono essere confermati.");

    order.Confirm();
    return Result<Guid>.Success(order.Id);
}

// ✅ Primitive wrapping — Quantity con validazione integrata
public readonly record struct Quantity
{
    public int Value { get; }
    private Quantity(int value) => Value = value;

    public static Quantity From(int value)
    {
        if (value <= 0)
            throw new DomainException("La quantità deve essere maggiore di zero.");
        return new Quantity(value);
    }
}
```

---

## Clean Code — Esempi C#

```csharp
// ❌ Magic number
if (elapsedSeconds > 86400) ExpireSession();

// ✅ Costante nominata
private const int SecondsInADay = 86400;
if (elapsedSeconds > SecondsInADay) ExpireSession();

// ❌ Condizione criptica
if (user.Age >= 18 && user.HasValidId && !user.IsBanned && sub.EndDate > DateTime.Now)
    GrantAccess();

// ✅ Condizioni espressive
bool isAdult = user.Age >= 18;
bool hasValidCredentials = user.HasValidId && !user.IsBanned;
bool hasActiveSubscription = sub.EndDate > DateTime.Now;
bool canAccess = isAdult && hasValidCredentials && hasActiveSubscription;

if (canAccess) GrantAccess();
```

---

## Testing — Template C#

La naming convention per i test è `MethodName_Condition_ExpectedResult()`.
La struttura è sempre Arrange → Act → Assert, senza commenti che lo dichiarano.

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
        _handler = new CreateOrderHandler(_db, Substitute.For<IOrderValidator>());
    }

    [Fact(DisplayName = "Ordine valido viene creato e restituisce Guid")]
    public async Task HandleAsync_ValidCommand_ReturnsSuccessWithGuid()
    {
        var command = new CreateOrderCommand(Guid.NewGuid(),
            [new OrderItemDto(Guid.NewGuid(), 2)]);

        var result = await _handler.HandleAsync(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
    }

    [Fact(DisplayName = "Ordine senza articoli restituisce Failure")]
    public async Task HandleAsync_EmptyItems_ReturnsFailure()
    {
        var command = new CreateOrderCommand(Guid.NewGuid(), []);

        var result = await _handler.HandleAsync(command, CancellationToken.None);

        result.IsSuccess.Should().BeFalse();
        result.Error.Should().NotBeNullOrEmpty();
    }
}
```

---

## Naming Conventions C#

```
Classi e interfacce:  PascalCase            → UserService, IUserRepository
Metodi e proprietà:   PascalCase            → GetUserById, FirstName
Variabili locali:     camelCase             → userName, totalCount
Campi privati:        _camelCase            → _logger, _repository
Costanti:             PascalCase            → MaxRetryCount, SecondsInADay
Enumerazioni:         PascalCase (nome e valori) → UserStatus.Active
Record/DTO:           PascalCase + suffisso → CreateOrderCommand, OrderSummaryDto
Test methods:         MethodName_Condition_ExpectedResult
```

---

## Regole .NET-Specific

Queste regole si aggiungono alle regole assolute dell'agente generico:

1. **MAI MediatR** — `ICommandHandler` / `IQueryHandler` sempre
2. **MAI `.Result` o `.Wait()`** — sempre `async/await`
3. **MAI entity complete in query di lettura** — `AsNoTracking()` + `Select()`
4. **MAI Controller-based** in progetti nuovi — solo Minimal API
5. **MAI lazy loading** — usare `Include()` esplicito o projection
6. **MAI `IQueryable` esposto** fuori dall'Infrastructure layer
7. **Sempre** `CancellationToken ct` in ogni metodo async
8. **Sempre** `Nullable enable` nel `.csproj`
9. **Sempre** `ImplicitUsings enable` nel `.csproj`
10. **Sempre** enum persistiti come stringa, mai come int
11. **MAI creare migrazioni** — solo suggerire il comando a fine implementazione