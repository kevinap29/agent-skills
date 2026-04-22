# EFCore .NET 10 Skill

## Description

Expert skill for Entity Framework Core on .NET 10 covering DbContext setup, migrations, repository patterns, query optimization, and SQLite for MAUI Hybrid. Use this skill whenever the user mentions EFCore, Entity Framework, DbContext, migrations, LINQ queries, repository pattern, Unit of Work, or database access in .NET 10 projects including MAUI Hybrid and Blazor Web. Always use this skill for DbContext configuration, model configuration via Fluent API, query patterns, seeding, and EFCore integration with CQRS handlers.

## NuGet Packages

```bash
# SQL Server (Web/API)
dotnet add package Microsoft.EntityFrameworkCore.SqlServer

# SQLite (MAUI Hybrid local DB)
dotnet add package Microsoft.EntityFrameworkCore.Sqlite

# Design (for migrations — dev only)
dotnet add package Microsoft.EntityFrameworkCore.Design

# Tools (global)
dotnet tool install --global dotnet-ef
```

## DbContext

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all IEntityTypeConfiguration<T> in assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

## Entity Configuration (Fluent API)

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Name)
            .IsRequired()
            .HasMaxLength(200);
        builder.Property(u => u.Email)
            .IsRequired()
            .HasMaxLength(320);
        builder.HasIndex(u => u.Email).IsUnique();

        builder.HasMany(u => u.Orders)
            .WithOne(o => o.User)
            .HasForeignKey(o => o.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

## Registration

### Web / ASP.NET Core (SQL Server)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sql => sql.EnableRetryOnFailure(3)
    ));
```

### MAUI Hybrid (SQLite)

```csharp
// MauiProgram.cs
var dbPath = Path.Combine(FileSystem.AppDataDirectory, "myapp.db");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite($"Data Source={dbPath}"));

// Run migrations on app start
var app = builder.Build();
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();
```

### Shared Extension Method

```csharp
// MyApp.Core/Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services,
        IConfiguration configuration,
        string? sqlitePath = null)
    {
        if (sqlitePath is not null)
        {
            services.AddDbContext<AppDbContext>(o =>
                o.UseSqlite($"Data Source={sqlitePath}"));
        }
        else
        {
            services.AddDbContext<AppDbContext>(o =>
                o.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
        }

        return services;
    }
}
```

## Migrations

```bash
# Add migration
dotnet ef migrations add InitialCreate --project MyApp.Core --startup-project MyApp.Web

# Apply to DB
dotnet ef database update --project MyApp.Core --startup-project MyApp.Web

# Generate SQL script
dotnet ef migrations script --idempotent -o migrations.sql
```

## Query Patterns with CQRS

### Read-only Query (no tracking)

```csharp
public class GetUserByIdQueryHandler : IRequestHandler<GetUserByIdQuery, UserDto?>
{
    private readonly AppDbContext _db;

    public GetUserByIdQueryHandler(AppDbContext db) => _db = db;

    public async Task<UserDto?> Handle(GetUserByIdQuery request, CancellationToken ct)
        => await _db.Users
            .AsNoTracking()                        // No change tracking for reads
            .Where(u => u.Id == request.Id)
            .Select(u => new UserDto(u.Id, u.Name, u.Email))
            .FirstOrDefaultAsync(ct);
}
```

### Paginated Query

```csharp
public record GetUsersQuery(int Page = 1, int PageSize = 20, string? Search = null)
    : IRequest<PagedResult<UserDto>>;

public class GetUsersQueryHandler : IRequestHandler<GetUsersQuery, PagedResult<UserDto>>
{
    private readonly AppDbContext _db;

    public GetUsersQueryHandler(AppDbContext db) => _db = db;

    public async Task<PagedResult<UserDto>> Handle(GetUsersQuery request, CancellationToken ct)
    {
        var query = _db.Users.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(request.Search))
            query = query.Where(u => u.Name.Contains(request.Search) ||
                                     u.Email.Contains(request.Search));

        var total = await query.CountAsync(ct);
        var items = await query
            .OrderBy(u => u.Name)
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(u => new UserDto(u.Id, u.Name, u.Email))
            .ToListAsync(ct);

        return new PagedResult<UserDto>(items, total, request.Page, request.PageSize);
    }
}
```

### Write Command with EFCore

```csharp
public class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, Guid>
{
    private readonly AppDbContext _db;
    private readonly IMediator _mediator;

    public CreateUserCommandHandler(AppDbContext db, IMediator mediator)
    {
        _db = db;
        _mediator = mediator;
    }

    public async Task<Guid> Handle(CreateUserCommand request, CancellationToken ct)
    {
        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = request.Name,
            Email = request.Email,
            CreatedAt = DateTimeOffset.UtcNow
        };

        _db.Users.Add(user);
        await _db.SaveChangesAsync(ct);

        await _mediator.Publish(new UserCreatedNotification(user.Id, user.Email), ct);

        return user.Id;
    }
}
```

## Entity Base Class

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTimeOffset CreatedAt { get; init; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? UpdatedAt { get; set; }
}
```

## Audit Interceptor (.NET 10)

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var entries = eventData.Context?.ChangeTracker
            .Entries<BaseEntity>()
            .Where(e => e.State is EntityState.Modified);

        if (entries is not null)
            foreach (var entry in entries)
                entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Register in DI
builder.Services.AddSingleton<AuditInterceptor>();
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString);
    options.AddInterceptors(sp.GetRequiredService<AuditInterceptor>());
});
```

## Seeding

```csharp
public static class DbSeeder
{
    public static async Task SeedAsync(AppDbContext db)
    {
        if (await db.Users.AnyAsync()) return;

        db.Users.AddRange(
            new User { Name = "Admin", Email = "admin@example.com" }
        );

        await db.SaveChangesAsync();
    }
}

// In Program.cs / MauiProgram.cs
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();
await DbSeeder.SeedAsync(db);
```

## Common EFCore .NET 10 Tips

- Use `AsNoTracking()` for all read-only queries — significant perf gain
- Prefer `Select()` projection over loading full entities for queries
- Use `ExecuteUpdateAsync` / `ExecuteDeleteAsync` for bulk operations (avoids loading entities)
- SQLite does not support all SQL Server features — avoid `DateTimeOffset` arithmetic in SQLite queries
- In MAUI Hybrid, always run `MigrateAsync()` on startup to handle schema updates after app updates
- Use `EnsureCreated()` only for SQLite dev/test — always use `MigrateAsync()` in production