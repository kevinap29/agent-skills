# Clean Architecture & DDD Skill (.NET 10)

## Description

Expert skill for designing and implementing Clean Architecture and Domain-Driven Design (DDD) in .NET 10. Focuses on separation of concerns, domain modeling, aggregate roots, value objects, and ensuring the core business logic remains independent of external frameworks. Use this skill when the user asks to structure a new project, refactor logic into the domain, or implement complex business rules.

## Core Principles

- **Independence**: Domain logic should not depend on UI, DB, or external libraries.
- **Dependency Rule**: Dependencies always point inwards (Infrastructure -> Application -> Domain).
- **Rich Domain Model**: Put logic inside entities and value objects, not just "Anemic" services.

## Layer Structure

```text
Solution/
├── src/
│   ├── MyApp.Domain/         # Entities, Value Objects, Domain Events, Repository Interfaces
│   ├── MyApp.Application/    # Use Cases (Commands/Queries), Handlers, DTOs, Interfaces
│   ├── MyApp.Infrastructure/ # DB Context, Repo Implementations, External Clients, Auth
│   └── MyApp.Web/            # Controllers, Minimal APIs, Blazor Components, Middleware
```

## DDD Building Blocks

### Value Object

```csharp
public abstract class ValueObject : IEquatable<ValueObject>
{
    public abstract IEnumerable<object> GetAtomicValues();

    public bool Equals(ValueObject? other) => other is not null && GetAtomicValues().SequenceEqual(other.GetAtomicValues());
    public override bool Equals(object? obj) => obj is ValueObject other && Equals(other);
    public override int GetHashCode() => GetAtomicValues().Aggregate(0, (acc, val) => acc ^ val.GetHashCode());
}

public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
    public override IEnumerable<object> GetAtomicValues() { yield return Amount; yield return Currency; }
}
```

### Entity & Aggregate Root

```csharp
public abstract class Entity
{
    public Guid Id { get; init; }
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void RaiseDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

public class Order : Entity // Aggregate Root
{
    public Guid CustomerId { get; private set; }
    public Money TotalAmount { get; private set; }
    public OrderStatus Status { get; private set; }

    public static Order Create(Guid customerId)
    {
        var order = new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Created };
        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }
}
```

## Application Layer (Use Case)

```csharp
public class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Result>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _uow;

    public PlaceOrderHandler(IOrderRepository repository, IUnitOfWork uow)
    {
        _repository = repository;
        _uow = uow;
    }

    public async Task<Result> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);
        _repository.Add(order);
        await _uow.SaveChangesAsync(ct);
        return Result.Success();
    }
}
```

## DDD Tips

- **Value Objects vs Entities**: Use Value Objects for things defined by their attributes (e.g., Address, Money). Use Entities for things defined by their identity.
- **Encapsulation**: Make setters `private` or `protected`. Use methods like `UpdateName()` to enforce business rules.
- **Aggregates**: Ensure an aggregate is the only way to modify its internal child entities (consistency boundary).
- **Inversion of Control**: Infrastructure implements interfaces defined in the Domain or Application layers.
- **Avoid Over-Engineering**: For simple CRUD, Clean Architecture might be overkill. Use Vertical Slices (Feature folders) instead.
