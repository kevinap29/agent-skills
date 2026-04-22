# Unit Testing & TDD Skill (.NET 10)

## Description

Expert skill for writing high-quality unit, integration, and architecture tests in .NET 10. Covers xUnit, NSubstitute for mocking, FluentAssertions for expressive assertions, and testing practices for CQRS, EFCore, and Web APIs. Use this skill when the user asks to write tests, setup a testing project, or improve code testability.

## Testing Stack

```bash
# Core testing framework
dotnet add package xunit
dotnet add package Microsoft.NET.Test.Sdk

# Mocking library
dotnet add package NSubstitute

# Fluent assertions
dotnet add package FluentAssertions

# Web API Integration Testing
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

## Unit Test Example (Service/Handler)

```csharp
public class CreateUserHandlerTests
{
    private readonly IUserRepository _repository = Substitute.For<IUserRepository>();
    private readonly CreateUserHandler _sut; // System Under Test

    public CreateUserHandlerTests()
    {
        _sut = new CreateUserHandler(_repository);
    }

    [Fact]
    public async Task Handle_Should_ReturnSuccess_When_DataIsValid()
    {
        // Arrange
        var command = new CreateUserCommand("John Doe", "john@example.com");
        _repository.IsEmailUniqueAsync(command.Email).Returns(true);

        // Act
        var result = await _sut.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        await _repository.Received(1).AddAsync(Arg.Any<User>());
    }

    [Fact]
    public async Task Handle_Should_ReturnFailure_When_EmailIsNotUnique()
    {
        // Arrange
        var command = new CreateUserCommand("John Doe", "john@example.com");
        _repository.IsEmailUniqueAsync(command.Email).Returns(false);

        // Act
        var result = await _sut.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Be(UserErrors.EmailNotUnique);
    }
}
```

## Integration Test (Web API)

```csharp
public class UserApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetUser_Should_ReturnOk_When_UserExists()
    {
        // Act
        var response = await _client.GetAsync("/api/users/1");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var user = await response.Content.ReadFromJsonAsync<UserDto>();
        user.Should().NotBeNull();
    }
}
```

## EFCore Testing (In-Memory / SQLite)

```csharp
public class DbTestBase
{
    protected AppDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite("DataSource=:memory:")
            .Options;

        var context = new AppDbContext(options);
        context.Database.OpenConnection();
        context.Database.EnsureCreated();
        return context;
    }
}
```

## Testing Tips

- **Naming Conventions**: Use `MethodName_Should_ExpectedBehavior_When_Scenario`.
- **AAA Pattern**: Always structure tests with **Arrange**, **Act**, and **Assert** sections.
- **Mocking**: Only mock external dependencies (DB, API, File System). Avoid mocking internal logic.
- **SUT**: Clearly mark the object being tested as `_sut`.
- **Architecture Tests**: Use `NetArchTest.eNet` to enforce architectural rules (e.g., "Domain should not depend on Infrastructure").
- **TDD Cycle**: Red (write failing test) -> Green (write code to pass) -> Refactor.
