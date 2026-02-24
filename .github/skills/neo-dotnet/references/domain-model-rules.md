# Domain Model Rules

## Entities

Entities have identity and lifecycle. They encapsulate state and behavior together.
State is protected — external code interacts through methods, never through setters.

```csharp
public sealed class Order : AggregateRoot
{
    // Private setter — state changes only through methods
    public OrderId Id { get; private init; }
    public CustomerId CustomerId { get; private init; }
    public Money TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTimeOffset CreatedAt { get; private init; }

    // Navigation — private collection, exposed as read-only
    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    // Private constructor — forces use of factory method
    private Order() { }

    // Factory method — enforces all invariants at creation
    public static Order Create(
        CustomerId customerId,
        IEnumerable<OrderItem> items,
        IOrderPricingService pricingService)
    {
        var itemList = items.ToList();

        // Domain invariant: an order must have at least one item
        if (itemList.Count == 0)
            throw new DomainRuleViolationException("Order must contain at least one item.");

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            CreatedAt = DateTimeOffset.UtcNow
        };

        foreach (var item in itemList)
            order._items.Add(item);

        order.TotalAmount = pricingService.CalculateTotal(order._items);

        // Raise domain event
        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id, customerId));

        return order;
    }

    // Behavioral method — encapsulates business rules
    public void Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleViolationException(
                $"Cannot confirm order in '{Status}' status. Only draft orders can be confirmed.");

        Status = OrderStatus.Confirmed;
        RaiseDomainEvent(new OrderConfirmedEvent(Id));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Completed)
            throw new DomainRuleViolationException(
                $"Cannot cancel order in '{Status}' status.");

        Status = OrderStatus.Cancelled;
        RaiseDomainEvent(new OrderCancelledEvent(Id, reason));
    }

    public void AddItem(OrderItem item, IOrderPricingService pricingService)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleViolationException(
                "Items can only be added to draft orders.");

        _items.Add(item);
        TotalAmount = pricingService.CalculateTotal(_items);
    }
}
```

## Strongly-Typed IDs

Every entity uses a strongly-typed ID to prevent primitive obsession and accidental
ID swapping between different entity types.

```csharp
// Record struct — value semantics, zero allocation
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value)
    {
        if (value == Guid.Empty)
            throw new DomainException("OrderId cannot be empty.");
        return new OrderId(value);
    }
    public override string ToString() => Value.ToString();
}

public readonly record struct CustomerId(Guid Value)
{
    public static CustomerId From(Guid value)
    {
        if (value == Guid.Empty)
            throw new DomainException("CustomerId cannot be empty.");
        return new CustomerId(value);
    }
}
```

## Value Objects

Value Objects have no identity. They are defined entirely by their attributes.
Use C# records for automatic equality-by-value. They are always immutable.

```csharp
// Simple value object
public sealed record Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email From(string value)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        if (!value.Contains('@') || value.Length > 256)
            throw new DomainException($"'{value}' is not a valid email address.");
        return new Email(value.ToLowerInvariant());
    }

    public override string ToString() => Value;
}

// Composite value object
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
            throw new DomainException("Amount cannot be negative.");
        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
            throw new DomainException("Currency must be a 3-letter ISO code.");
        return new Money(amount, currency.ToUpperInvariant());
    }

    public static Money Zero(string currency) => Create(0, currency);

    // Value objects contain behavior relevant to their value
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"Cannot add {Currency} and {other.Currency}.");
        return Create(Amount + other.Amount, Currency);
    }

    public Money Multiply(int quantity)
        => Create(Amount * quantity, Currency);
}

// Address value object
public sealed record Address(
    string Street,
    string City,
    string PostalCode,
    string Country)
{
    // Validation in a static factory when rules are complex
    public static Address Create(string street, string city, string postalCode, string country)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(street);
        ArgumentException.ThrowIfNullOrWhiteSpace(city);
        ArgumentException.ThrowIfNullOrWhiteSpace(postalCode);
        ArgumentException.ThrowIfNullOrWhiteSpace(country);
        return new Address(street, city, postalCode, country);
    }
}

// Quantity value object — wraps primitive with rules
public readonly record struct Quantity
{
    public int Value { get; }
    private Quantity(int value) => Value = value;

    public static Quantity From(int value)
    {
        if (value <= 0)
            throw new DomainException("Quantity must be greater than zero.");
        return new Quantity(value);
    }
}
```

## Aggregate Design

An Aggregate is a cluster of entities and value objects with a defined boundary.
The Aggregate Root is the only entity that external code can reference. All
modifications go through the root.

```csharp
// Base class for aggregate roots
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = [];
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Design rules:
// 1. Only the Aggregate Root is referenced from outside the boundary
// 2. Child entities (e.g., OrderItem) have no public repository
// 3. Transactions align with aggregate boundaries — one aggregate per transaction
// 4. Cross-aggregate references use IDs only, never navigation properties
// 5. Keep aggregates small — prefer separate aggregates connected by ID
```

## Domain Events

Domain Events represent something meaningful that happened in the domain. They are
raised by aggregates and dispatched after persistence (to ensure consistency).

```csharp
// Marker interface — Domain layer
public interface IDomainEvent
{
    DateTimeOffset OccurredAt { get; }
}

// Concrete event — Domain layer
public sealed record OrderCreatedEvent(
    OrderId OrderId,
    CustomerId CustomerId) : IDomainEvent
{
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}

public sealed record OrderConfirmedEvent(OrderId OrderId) : IDomainEvent
{
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}

// Event handler — Application layer
public sealed class OrderConfirmedEventHandler(
    IEmailService emailService,
    IOrderRepository orderRepository)
{
    public async Task HandleAsync(OrderConfirmedEvent @event, CancellationToken ct)
    {
        var order = await orderRepository.GetByIdAsync(@event.OrderId.Value, ct);
        if (order is null) return;

        await emailService.SendOrderConfirmationAsync(order.CustomerId, order.Id, ct);
    }
}
```

## Domain Services

Domain Services contain business logic that spans multiple aggregates or doesn't
naturally belong to a single entity. They are stateless and injected via DI.

```csharp
// Interface in Domain layer
public interface IOrderPricingService
{
    Money CalculateTotal(IReadOnlyCollection<OrderItem> items);
}

// Implementation can be in Domain (if no external dependencies)
// or Infrastructure (if it needs external data like tax rates)
public sealed class OrderPricingService : IOrderPricingService
{
    public Money CalculateTotal(IReadOnlyCollection<OrderItem> items)
    {
        if (items.Count == 0)
            return Money.Zero("EUR");

        return items
            .Select(item => item.UnitPrice.Multiply(item.Quantity.Value))
            .Aggregate((a, b) => a.Add(b));
    }
}
```

## Enumeration Pattern

For domain enums that carry behavior or need persistence stability, use the
enumeration class pattern instead of C# enums.

```csharp
// Simple C# enum is fine for pure status flags
public enum OrderStatus
{
    Draft,
    Confirmed,
    Shipped,
    Completed,
    Cancelled
}

// Use enumeration class when you need behavior attached to values
public abstract class PaymentMethod
{
    public static readonly PaymentMethod CreditCard = new CreditCardPayment();
    public static readonly PaymentMethod BankTransfer = new BankTransferPayment();

    public abstract string Code { get; }
    public abstract decimal CalculateFee(Money amount);

    private sealed class CreditCardPayment : PaymentMethod
    {
        public override string Code => "CC";
        public override decimal CalculateFee(Money amount) => amount.Amount * 0.029m;
    }

    private sealed class BankTransferPayment : PaymentMethod
    {
        public override string Code => "BT";
        public override decimal CalculateFee(Money amount) => 1.50m;
    }
}
```
