# CQRS Pattern Skill (.NET 10)

## Description

Expert skill for designing and implementing CQRS (Command Query Responsibility Segregation) architecture in .NET 10 projects covering domain modeling, Result pattern, error handling, Feature Slicing (Vertical Slice Architecture), validation with FluentValidation, domain events, and integration with EFCore and Blazor/MAUI UI. Use this skill whenever the user asks about CQRS architecture, how to structure Commands vs Queries, Result-T return types, domain events, FluentValidation in handlers, Feature Slice folder structure, or how to wire CQRS into Blazor components and MAUI Hybrid apps. Always prefer this skill over ad-hoc patterns when the user is building any non-trivial .NET 10 application with business logic separation.

## Overview

CQRS separates **write** operations (Commands) from **read** operations (Queries).  
This skill covers the **architectural patterns** on top of any mediator library.  
For mediator library selection (MediatR v12, FreeMediator, Mediator source-gen), see `mediatr-free-cqrs` skill.

```
UI (Blazor / MAUI)
      │
      ▼
  IMediator.Send()
      │
   ┌──┴──────────────────────────────────────┐
   │  Command Side          Query Side        │
   │  (writes, mutations)   (reads, projections) │
   │  ─────────────         ────────────      │
   │  Handler → EFCore      Handler → EFCore  │
   │  → Domain Events       → DTOs / ViewModels│
   └─────────────────────────────────────────┘
```

> For detailed reference files, see:
> - `references/result-pattern.md` — Result<T>, error types, error handling
> - `references/feature-slicing.md` — Vertical Slice folder structure & conventions
> - `references/domain-events.md` — Domain events, outbox pattern
> - `references/blazor-integration.md` — Wiring CQRS to Blazor/MAUI components

---

## Core Contracts

### Result Pattern (always use instead of throwing exceptions for business errors)

```csharp
// MyApp.Core/Common/Result.cs
public class Result<T>
{
    public T? Value { get; }
    public bool IsSuccess { get; }
    public Error Error { get; }

    private Result(T value) { Value = value; IsSuccess = true; Error = Error.None; }
    private Result(Error error) { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(Error error) => new(error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error);
}

public class Result
{
    public bool IsSuccess { get; }
    public Error Error { get; }

    protected Result(bool isSuccess, Error error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static implicit operator Result(Error error) => Failure(error);
}
```

### Error Type

```csharp
// MyApp.Core/Common/Error.cs
public record Error(string Code, string Description, ErrorType Type = ErrorType.Failure)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.None);
    public static readonly Error NullValue = new("General.Null", "Null value was provided", ErrorType.Failure);

    public static Error NotFound(string resource, object id)
        => new($"{resource}.NotFound", $"{resource} with id '{id}' was not found.", ErrorType.NotFound);

    public static Error Conflict(string resource, string detail)
        => new($"{resource}.Conflict", detail, ErrorType.Conflict);

    public static Error Validation(string field, string message)
        => new($"Validation.{field}", message, ErrorType.Validation);
}

public enum ErrorType { None, Failure, NotFound, Conflict, Validation, Unauthorized }
```

---

## Command Pattern

### Command Definition

```csharp
// Naming: <Verb><Entity>Command
// Returns Result or Result<T> — never throws for business errors

// Command with return value
public record CreateProductCommand(
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    Guid CategoryId
) : IRequest<Result<Guid>>;

// Command without return value
public record DeleteProductCommand(Guid ProductId) : IRequest<Result>;

// Command with complex return
public record UpdateProductCommand(
    Guid ProductId,
    string Name,
    decimal Price
) : IRequest<Result<ProductDto>>;
```

### Command Handler

```csharp
public class CreateProductCommandHandler
    : IRequestHandler<CreateProductCommand, Result<Guid>>
{
    private readonly AppDbContext _db;
    private readonly IPublisher _publisher;  // IMediator or IPublisher

    public CreateProductCommandHandler(AppDbContext db, IPublisher publisher)
    {
        _db = db;
        _publisher = publisher;
    }

    public async Task<Result<Guid>> Handle(
        CreateProductCommand request,
        CancellationToken ct)
    {
        // 1. Guard / business rule check
        var categoryExists = await _db.Categories
            .AnyAsync(c => c.Id == request.CategoryId, ct);

        if (!categoryExists)
            return Result<Guid>.Failure(
                Error.NotFound(nameof(Category), request.CategoryId));

        // 2. Duplicate check
        var nameExists = await _db.Products
            .AnyAsync(p => p.Name == request.Name, ct);

        if (nameExists)
            return Result<Guid>.Failure(
                Error.Conflict(nameof(Product), $"Product '{request.Name}' already exists."));

        // 3. Create entity
        var product = Product.Create(
            request.Name,
            request.Description,
            request.Price,
            request.StockQuantity,
            request.CategoryId);

        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);

        // 4. Publish domain event
        await _publisher.Publish(
            new ProductCreatedNotification(product.Id, product.Name), ct);

        return Result<Guid>.Success(product.Id);
    }
}
```

### Command Validator (FluentValidation)

```csharp
// Always co-locate validator with command
public class CreateProductCommandValidator
    : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0.")
            .LessThanOrEqualTo(999_999).WithMessage("Price is too large.");

        RuleFor(x => x.StockQuantity)
            .GreaterThanOrEqualTo(0).WithMessage("Stock quantity cannot be negative.");

        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("Category is required.");
    }
}
```

---

## Query Pattern

### Query Definition

```csharp
// Naming: Get<Entity>Query / Get<Entity>ByXQuery / List<Entity>sQuery

// Single item
public record GetProductByIdQuery(Guid ProductId) : IRequest<Result<ProductDto>>;

// List with filtering
public record ListProductsQuery(
    int Page = 1,
    int PageSize = 20,
    string? Search = null,
    Guid? CategoryId = null,
    ProductSortBy SortBy = ProductSortBy.Name
) : IRequest<Result<PagedList<ProductDto>>>;
```

### Query Handler (always use AsNoTracking + projection)

```csharp
public class GetProductByIdQueryHandler
    : IRequestHandler<GetProductByIdQuery, Result<ProductDto>>
{
    private readonly AppDbContext _db;

    public GetProductByIdQueryHandler(AppDbContext db) => _db = db;

    public async Task<Result<ProductDto>> Handle(
        GetProductByIdQuery request,
        CancellationToken ct)
    {
        var product = await _db.Products
            .AsNoTracking()
            .Where(p => p.Id == request.ProductId)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Description,
                p.Price,
                p.StockQuantity,
                p.Category.Name,
                p.CreatedAt))
            .FirstOrDefaultAsync(ct);

        return product is null
            ? Result<ProductDto>.Failure(Error.NotFound(nameof(Product), request.ProductId))
            : Result<ProductDto>.Success(product);
    }
}
```

### Paginated List Query Handler

```csharp
public class ListProductsQueryHandler
    : IRequestHandler<ListProductsQuery, Result<PagedList<ProductDto>>>
{
    private readonly AppDbContext _db;

    public ListProductsQueryHandler(AppDbContext db) => _db = db;

    public async Task<Result<PagedList<ProductDto>>> Handle(
        ListProductsQuery request,
        CancellationToken ct)
    {
        var query = _db.Products.AsNoTracking();

        // Filter
        if (!string.IsNullOrWhiteSpace(request.Search))
            query = query.Where(p =>
                p.Name.Contains(request.Search) ||
                p.Description.Contains(request.Search));

        if (request.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == request.CategoryId.Value);

        // Sort
        query = request.SortBy switch
        {
            ProductSortBy.Name => query.OrderBy(p => p.Name),
            ProductSortBy.Price => query.OrderBy(p => p.Price),
            ProductSortBy.Newest => query.OrderByDescending(p => p.CreatedAt),
            _ => query.OrderBy(p => p.Name)
        };

        // Project + paginate
        var total = await query.CountAsync(ct);
        var items = await query
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductDto(
                p.Id, p.Name, p.Description,
                p.Price, p.StockQuantity,
                p.Category.Name, p.CreatedAt))
            .ToListAsync(ct);

        return Result<PagedList<ProductDto>>.Success(
            new PagedList<ProductDto>(items, total, request.Page, request.PageSize));
    }
}
```

---

## DTOs & Shared Types

```csharp
// Immutable records for query results
public record ProductDto(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    int StockQuantity,
    string CategoryName,
    DateTimeOffset CreatedAt);

// Pagination wrapper
public record PagedList<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}
```

---

## Feature Slice Folder Structure

See `references/feature-slicing.md` for full conventions.

Quick summary:
```
MyApp.Core/Features/
├── Products/
│   ├── Commands/
│   │   ├── CreateProduct/
│   │   │   ├── CreateProductCommand.cs
│   │   │   ├── CreateProductCommandHandler.cs
│   │   │   └── CreateProductCommandValidator.cs
│   │   └── DeleteProduct/
│   ├── Queries/
│   │   ├── GetProductById/
│   │   └── ListProducts/
│   ├── Events/
│   │   └── ProductCreatedNotification.cs
│   └── Dtos/
│       └── ProductDto.cs
```

---

## Domain Events Pattern

See `references/domain-events.md` for full outbox pattern.

Quick summary — raise events from aggregate root, dispatch after SaveChanges:

```csharp
// Domain event (INotification)
public record ProductCreatedNotification(Guid ProductId, string Name) : INotification;

// Handler
public class ProductCreatedNotificationHandler
    : INotificationHandler<ProductCreatedNotification>
{
    public async Task Handle(ProductCreatedNotification n, CancellationToken ct)
    {
        // Send email, update cache, trigger external service, etc.
    }
}
```

---

## Blazor / MAUI Integration

See `references/blazor-integration.md` for full component patterns.

Quick summary:

```razor
@inject IMediator Mediator

@code {
    private async Task HandleSubmit()
    {
        var result = await Mediator.Send(new CreateProductCommand(...));

        result.Match(
            onSuccess: id => NavigationManager.NavigateTo($"/products/{id}"),
            onFailure: error => ShowError(error.Description)
        );
    }
}
```

---

## Rules & Conventions

| Rule | Why |
|------|-----|
| Commands return `Result` or `Result<T>`, never raw types | Consistent error handling without exceptions |
| Queries use `AsNoTracking()` + `Select()` projection | Performance — no entity materialization |
| Validators are co-located with their Command | Single responsibility, easy to find |
| Domain events are published AFTER `SaveChangesAsync` | Consistency — event only fires if write succeeded |
| DTOs are immutable `record` types | Thread-safe, value equality, pattern matching |
| Handlers have no business logic in the constructor | DI-friendly, easier to test |
| Commands use past-tense in events (`ProductCreated`) | Domain event naming convention |