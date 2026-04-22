# ASP.NET Core Web API Skill (.NET 10)

## Description

Expert skill for building modern, high-performance Web APIs using ASP.NET Core .NET 10. Focuses on Minimal APIs, modern middleware configuration, JWT authentication, OpenAPI (Swagger), and industry-standard security practices. Use this skill when the user asks to create API endpoints, configure authentication/authorization, add custom middleware, or setup API documentation.

## Minimal API Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddOpenApi(); // .NET 10 OpenAPI support
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

var app = builder.Build();

// Configure Middleware
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Map Endpoints
app.MapGet("/api/hello", () => Results.Ok(new { Message = "Hello World" }))
   .WithName("GetHello")
   .WithOpenApi();

app.Run();
```

## Authentication & Authorization (JWT)

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});
```

## Custom Middleware (Exception Handling)

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception has occurred.");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new { Error = "Internal Server Error" });
        }
    }
}

// Register
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

## Validation (FluentValidation Integration)

```csharp
// Define Validator
public class UserValidator : AbstractValidator<UserDto>
{
    public UserValidator()
    {
        RuleFor(x => x.Email).EmailAddress();
        RuleFor(x => x.Age).InclusiveBetween(18, 99);
    }
}

// Endpoint with Validation
app.MapPost("/api/users", async (UserDto user, IValidator<UserDto> validator) =>
{
    var validationResult = await validator.ValidateAsync(user);
    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }
    return Results.Created($"/api/users/{user.Id}", user);
});
```

## API Tips

- **Use `Results`**: In Minimal APIs, return `Results.Ok()`, `Results.NotFound()`, etc., for consistent status codes.
- **Dependency Injection**: Use `[FromServices]` or just add the service to the lambda parameters in `MapGet/MapPost`.
- **Typed Results**: Use `TypedResults<T>` for better unit testing and OpenAPI documentation.
- **Grouping**: Use `app.MapGroup("/api/v1/products")` to organize related endpoints and apply common authorization.
- **OpenAPI**: .NET 10 moves towards `Microsoft.AspNetCore.OpenApi`. Ensure `AddOpenApi()` is called to generate documentation.
