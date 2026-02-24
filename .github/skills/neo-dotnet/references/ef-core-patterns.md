# EF Core Patterns for Clean Architecture

## DbContext Design

One DbContext per bounded context. The DbContext lives in the Infrastructure layer
and implements the IUnitOfWork interface from Domain.

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options), IUnitOfWork
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all IEntityTypeConfiguration<T> from this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }

    // Dispatch domain events before saving (optional pattern)
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Optionally dispatch domain events here
        return await base.SaveChangesAsync(ct);
    }
}
```

## Entity Configuration

Each entity has its own configuration class implementing `IEntityTypeConfiguration<T>`.
All configurations live in `Infrastructure/Persistence/Configurations/`.

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        // Strongly-typed ID conversion
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => OrderId.From(value));

        // Value Object as owned type
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("TotalAmount")
                .HasPrecision(18, 2);
            money.Property(m => m.Currency)
                .HasColumnName("Currency")
                .HasMaxLength(3);
        });

        // Enum conversion
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        // One-to-many inside aggregate
        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("OrderId")
            .OnDelete(DeleteBehavior.Cascade);

        // Relationship to another aggregate (by ID only)
        builder.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, value => CustomerId.From(value));

        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.CreatedAt);
    }
}
```

## Repository Pattern

Repository interfaces live in Domain, implementations in Infrastructure. The generic
base provides CRUD, concrete repositories add aggregate-specific queries.

```csharp
// Domain layer — generic interface
public interface IRepository<T> where T : AggregateRoot
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Remove(T entity);
}

// Domain layer — aggregate-specific interface (extends generic)
public interface IOrderRepository : IRepository<Order>
{
    Task<Order?> GetWithItemsAsync(OrderId id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(CustomerId customerId, CancellationToken ct = default);
}

// Infrastructure layer — generic implementation
public abstract class RepositoryBase<T>(AppDbContext context)
    : IRepository<T> where T : AggregateRoot
{
    protected AppDbContext Context { get; } = context;

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await Context.Set<T>().FindAsync([id], ct);

    public async Task AddAsync(T entity, CancellationToken ct = default)
        => await Context.Set<T>().AddAsync(entity, ct);

    public void Remove(T entity)
        => Context.Set<T>().Remove(entity);
}

// Infrastructure layer — concrete implementation
public sealed class OrderRepository(AppDbContext context)
    : RepositoryBase<Order>(context), IOrderRepository
{
    public async Task<Order?> GetWithItemsAsync(OrderId id, CancellationToken ct = default)
        => await Context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(
        CustomerId customerId,
        CancellationToken ct = default)
        => await Context.Orders
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);
}
```

## Query Patterns

### Read-Only Queries

For read operations, use `AsNoTracking()` and project directly to DTOs with `Select()`.
Query handlers can inject `AppDbContext` directly — they don't need to go through
repositories since they don't modify state.

```csharp
// Projection to DTO — always use Select, never load full entities for reads
var result = await context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Active)
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id.Value,
        CustomerName = o.Customer.Name,
        Total = o.TotalAmount.Amount,
        ItemCount = o.Items.Count
    })
    .ToListAsync(ct);
```

### No IQueryable Leaking

Never expose `IQueryable` outside the Infrastructure layer. Repository methods
return `Task<T>`, `Task<T?>`, or `Task<IReadOnlyList<T>>`.

### Pagination

```csharp
public async Task<PagedResult<OrderSummaryDto>> GetPagedAsync(
    int page, int pageSize, CancellationToken ct = default)
{
    var query = context.Orders.AsNoTracking();

    var totalCount = await query.CountAsync(ct);

    var items = await query
        .OrderByDescending(o => o.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderSummaryDto { /* ... */ })
        .ToListAsync(ct);

    return new PagedResult<OrderSummaryDto>(items, totalCount, page, pageSize);
}
```

### Specification Pattern (for complex queries)

```csharp
// Domain layer — specification base
public abstract class Specification<T> where T : class
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
        => ToExpression().Compile()(entity);
}

// Domain layer — concrete specification
public sealed class ActiveOrdersForCustomerSpec(CustomerId customerId)
    : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => order => order.CustomerId == customerId
            && order.Status == OrderStatus.Active;
}

// Repository usage
public async Task<IReadOnlyList<Order>> GetBySpecAsync(
    Specification<Order> spec, CancellationToken ct = default)
    => await Context.Orders
        .Where(spec.ToExpression())
        .ToListAsync(ct);
```

## Migration Conventions

Use descriptive migration names that explain what changed, not when.

```bash
# Good — describes the change
dotnet ef migrations add AddOrderStatusIndex
dotnet ef migrations add CreateCustomerEmailUniqueConstraint
dotnet ef migrations add SplitAddressIntoValueObject

# Bad — meaningless names
dotnet ef migrations add Update1
dotnet ef migrations add Migration_20250101
```

Always review generated migrations before applying. Never auto-apply in production.

## Seeding

Use `IEntityTypeConfiguration` or a dedicated `IDataSeeder` interface for test/dev data.
Production seed data goes in migration files.

```csharp
// In entity configuration (for reference/lookup data)
builder.HasData(
    OrderStatus.Draft,
    OrderStatus.Active,
    OrderStatus.Completed,
    OrderStatus.Cancelled);

// For dev/test seeding, use a separate service
public interface IDataSeeder
{
    Task SeedAsync(CancellationToken ct = default);
}
```

## Performance Rules

These rules apply to all EF Core usage in the project:

1. Always use `AsNoTracking()` for read-only queries.
2. Always use `Select()` to project to DTOs — never return full entities from queries.
3. Use explicit `Include()` for eager loading — never rely on lazy loading.
4. Use `AsSplitQuery()` for queries with multiple collection includes.
5. Always pass `CancellationToken` to all async methods.
6. Use `FindAsync` for single-entity lookups by primary key.
7. Batch operations with `ExecuteUpdateAsync` / `ExecuteDeleteAsync` for bulk changes.
8. Add indexes for frequently queried columns in entity configurations.
