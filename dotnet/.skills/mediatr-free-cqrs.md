# MediatR-Free CQRS Skill (.NET 10)

## Description

Expert skill for implementing CQRS and Mediator patterns in .NET using free/open-source alternatives to MediatR v13+ (which went commercial). Use this skill whenever the user mentions MediatR, CQRS, Command/Query separation, IRequest, IRequestHandler, Pipeline Behaviors, or mediator pattern in .NET. Covers MediatR v12.x (last MIT version), Mediator (source generator), FreeMediator, and Wolverine as free options. Always use this skill when deciding which mediator library to use, setting up CQRS handlers, pipeline behaviors, notifications, and integrating with EFCore and dependency injection.

## Context: MediatR Licensing

MediatR v13+ (by LuckyPennySoftware) is now dual-licensed under RPL-1.5 / Commercial. Free only for:
- Companies under $5M revenue
- Non-profits & education
- Non-production environments
- A license key is required even for the free Community tier

**Last fully free (MIT) version: MediatR v12.4.x**

---

## Free Options Comparison

| Library | NuGet | License | MediatR Compat | Performance |
|---------|-------|---------|----------------|-------------|
| **MediatR v12.4.1** | `MediatR` 12.4.1 | MIT | ✅ IS MediatR | Standard |
| **Mediator** | `Mediator` | MIT | ~90% API compat | ⚡ Faster (source gen) |
| **FreeMediator** | `FreeMediator` | MIT | ✅ Drop-in replacement | Standard |
| **Wolverine** | `WolverineFx` | MIT | ❌ Different API | ⚡⚡ Fastest |

---

## Option 1: MediatR v12.4.1 (Recommended — last MIT, most compatible)

```bash
dotnet add package MediatR --version 12.4.1
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection --version 12.4.0
```

### Registration (Program.cs / ServiceExtensions)

```csharp
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(AssemblyMarker).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
});
```

### Command (no return value)

```csharp
// Command definition
public record CreateUserCommand(string Name, string Email) : IRequest<Guid>;

// Handler
public class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, Guid>
{
    private readonly AppDbContext _db;

    public CreateUserCommandHandler(AppDbContext db) => _db = db;

    public async Task<Guid> Handle(CreateUserCommand request, CancellationToken ct)
    {
        var user = new User { Name = request.Name, Email = request.Email };
        _db.Users.Add(user);
        await _db.SaveChangesAsync(ct);
        return user.Id;
    }
}
```

### Query

```csharp
// Query definition
public record GetUserByIdQuery(Guid Id) : IRequest<UserDto?>;

// Handler
public class GetUserByIdQueryHandler : IRequestHandler<GetUserByIdQuery, UserDto?>
{
    private readonly AppDbContext _db;

    public GetUserByIdQueryHandler(AppDbContext db) => _db = db;

    public async Task<UserDto?> Handle(GetUserByIdQuery request, CancellationToken ct)
        => await _db.Users
            .Where(u => u.Id == request.Id)
            .Select(u => new UserDto(u.Id, u.Name, u.Email))
            .FirstOrDefaultAsync(ct);
}
```

### Notification (fan-out events)

```csharp
public record UserCreatedNotification(Guid UserId, string Email) : INotification;

public class SendWelcomeEmailHandler : INotificationHandler<UserCreatedNotification>
{
    public async Task Handle(UserCreatedNotification notification, CancellationToken ct)
    {
        // Send welcome email
    }
}

public class AuditLogHandler : INotificationHandler<UserCreatedNotification>
{
    public async Task Handle(UserCreatedNotification notification, CancellationToken ct)
    {
        // Write audit log
    }
}
```

### Dispatching in Blazor / MAUI

```razor
@inject IMediator Mediator

@code {
    private async Task CreateUser()
    {
        var id = await Mediator.Send(new CreateUserCommand(name, email));
        await Mediator.Publish(new UserCreatedNotification(id, email));
    }

    private async Task LoadUser(Guid id)
    {
        user = await Mediator.Send(new GetUserByIdQuery(id));
    }
}
```

---

## Pipeline Behaviors (Middleware)

### Validation Behavior (with FluentValidation)

```csharp
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

### Logging Behavior

```csharp
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        _logger.LogInformation("Handling {RequestName}", typeof(TRequest).Name);
        var response = await next();
        _logger.LogInformation("Handled {RequestName}", typeof(TRequest).Name);
        return response;
    }
}
```

### Performance Behavior

```csharp
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly Stopwatch _timer = new();
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;

    public PerformanceBehavior(ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        _timer.Restart();
        var response = await next();
        _timer.Stop();

        if (_timer.ElapsedMilliseconds > 500)
            _logger.LogWarning("Slow request: {Name} ({Elapsed}ms)",
                typeof(TRequest).Name, _timer.ElapsedMilliseconds);

        return response;
    }
}
```

---

## Option 2: FreeMediator (Drop-in replacement, MIT)

```bash
dotnet add package FreeMediator
```

API is identical to MediatR. Just change namespace if needed:

```csharp
// Replace: using MediatR;
using FreeMediator;

// Everything else stays the same:
// IRequest<T>, IRequestHandler<TRequest, TResponse>, IMediator, etc.
```

---

## Option 3: Mediator (Source Generator, fastest, MIT)

```bash
dotnet add package Mediator
```

Zero reflection — generates dispatch code at compile time:

```csharp
// Registration
builder.Services.AddMediator();

// Handler — same concept, slightly different interface
public class CreateUserHandler : ICommandHandler<CreateUserCommand, Guid>
{
    public ValueTask<Guid> Handle(CreateUserCommand command, CancellationToken ct)
    {
        // ...
    }
}

// Dispatch
@inject IMediator Mediator
var id = await Mediator.Send(new CreateUserCommand(name, email));
```

---

## Folder Structure (CQRS)

```
MyApp.Core/
├── Features/
│   ├── Users/
│   │   ├── Commands/
│   │   │   ├── CreateUser/
│   │   │   │   ├── CreateUserCommand.cs
│   │   │   │   ├── CreateUserCommandHandler.cs
│   │   │   │   └── CreateUserCommandValidator.cs
│   │   ├── Queries/
│   │   │   ├── GetUserById/
│   │   │   │   ├── GetUserByIdQuery.cs
│   │   │   │   └── GetUserByIdQueryHandler.cs
│   │   └── Events/
│   │       ├── UserCreatedNotification.cs
│   │       └── UserCreatedNotificationHandlers.cs
├── Common/
│   ├── Behaviors/
│   │   ├── ValidationBehavior.cs
│   │   ├── LoggingBehavior.cs
│   │   └── PerformanceBehavior.cs
│   └── Exceptions/
│       └── ValidationException.cs
```

## Decision Guide

```
Is your company under $5M revenue or non-commercial?
  → MediatR v12.4.1 (MIT, familiar API, no risk)
  OR MediatR v13+ Community (free with license key)

Do you need maximum performance / compile-time safety?
  → Mediator (source generator)

Need an exact drop-in with zero code changes?
  → FreeMediator

Building event-driven / message-based architecture?
  → Wolverine (different API but powerful)
```