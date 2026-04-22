# Docker & DevOps Skill (.NET 10)

## Description

Expert skill for containerizing .NET 10 applications and setting up CI/CD pipelines. Covers multi-stage Dockerfiles, Docker Compose for local development, and GitHub Actions for automated build/test/deploy. Use this skill when the user needs to deploy their app, setup Docker, or automate their development workflow.

## .NET 10 Multi-Stage Dockerfile

```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj and restore as distinct layers
COPY ["MyApp.Web/MyApp.Web.csproj", "MyApp.Web/"]
COPY ["MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
RUN dotnet restore "MyApp.Web/MyApp.Web.csproj"

# Copy everything else and build
COPY . .
WORKDIR "/src/MyApp.Web"
RUN dotnet build "MyApp.Web.csproj" -c Release -o /app/build

# Publish Stage
FROM build AS publish
RUN dotnet publish "MyApp.Web.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime Stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Web.dll"]
```

## Docker Compose (Local Dev with DB)

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=password
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
```

## GitHub Actions (.NET Build & Test)

```yaml
name: .NET CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 10.0.x
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore -c Release
    
    - name: Test
      run: dotnet test --no-build --verbosity normal
```

## Configuration & Environments

```csharp
// Program.cs
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables();

// Use Options Pattern
builder.Services.Configure<AppOptions>(
    builder.Configuration.GetSection(AppOptions.SectionName));
```

## Docker & DevOps Tips

- **Layer Caching**: Copy `.csproj` files and run `dotnet restore` *before* copying the rest of the source code to speed up Docker builds.
- **Environment Variables**: Use `__` (double underscore) as a separator in environment variables to override nested configuration values (e.g., `ConnectionStrings__Default`).
- **Small Images**: Use `aspnet` runtime image for the final stage, not the full `sdk`.
- **Health Checks**: Add `builder.Services.AddHealthChecks()` and a `HealthCheck` instruction in your Dockerfile to ensure container reliability.
- **Secrets**: Never commit secrets to source control. Use environment variables, User Secrets for local dev, or Key Vaults in production.
- **Multi-Platform**: Use `docker buildx` to build images for both x64 and ARM64 (for Apple Silicon and modern cloud servers).
