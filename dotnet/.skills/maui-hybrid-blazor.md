# .NET 10 MAUI Hybrid Blazor Skill

## Description

Expert skill for building .NET 10 MAUI Hybrid Blazor applications combining native mobile/desktop capabilities with Blazor web UI. Use this skill whenever the user mentions MAUI Hybrid, BlazorWebView, cross-platform app with Blazor, MAUI + Blazor, native device access in Blazor, or asks about sharing components between MAUI Hybrid and a Blazor Web app. Always use this skill for project structure, render mode decisions, platform services, JS interop in MAUI context, and shared RCL architecture.

## Overview

MAUI Hybrid uses `BlazorWebView` to host Razor components inside a native MAUI shell. The Blazor UI runs 
in-process (not WebAssembly), giving direct access to .NET and native platform APIs with no SignalR or 
network required.

## Project Structure (Recommended)

```
Solution/
├── MyApp.Shared/              # Razor Class Library (RCL) — shared components & pages
│   ├── Components/
│   ├── Pages/
│   ├── Services/              # Interfaces only
│   └── _Imports.razor
├── MyApp.MAUI/                # .NET 10 MAUI Hybrid project
│   ├── Platforms/
│   ├── MauiProgram.cs
│   ├── MainPage.xaml
│   └── wwwroot/
├── MyApp.Web/                 # ASP.NET Core Blazor Web (optional, shares RCL)
│   ├── Program.cs
│   └── Components/
└── MyApp.Core/                # Domain logic, EFCore, MediatR handlers
```

## MauiProgram.cs Bootstrap (.NET 10)

```csharp
using Microsoft.Extensions.Logging;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();

        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
            });

        builder.Services.AddMauiBlazorWebView();

#if DEBUG
        builder.Services.AddBlazorWebViewDeveloperTools();
        builder.Logging.AddDebug();
#endif

        // Register platform services
        builder.Services.AddSingleton<IDeviceService, MauiDeviceService>();

        // Register shared app services (EFCore, MediatR, etc.)
        builder.Services.AddApplicationServices(builder.Configuration);

        return builder.Build();
    }
}
```

## MainPage.xaml — BlazorWebView

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:blazor="clr-namespace:Microsoft.AspNetCore.Components.WebView.Maui;assembly=Microsoft.AspNetCore.Components.WebView.Maui"
             x:Class="MyApp.MAUI.MainPage">
    <blazor:BlazorWebView HostPage="wwwroot/index.html">
        <blazor:BlazorWebView.RootComponents>
            <blazor:RootComponent Selector="#app" ComponentType="{x:Type shared:Routes}" />
        </blazor:BlazorWebView.RootComponents>
    </blazor:BlazorWebView>
</ContentPage>
```

## wwwroot/index.html (MAUI)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover" />
    <title>MyApp</title>
    <base href="/" />
    <link rel="stylesheet" href="css/app.css" />
    <link rel="stylesheet" href="MyApp.Shared.styles.css" />
</head>
<body>
    <div id="app">Loading...</div>
    <script src="_framework/blazor.webview.js" autostart="false"></script>
    <script>
        Blazor.start();
    </script>
</body>
</html>
```

## Platform Service Pattern

Define interfaces in the RCL, implement per-platform in MAUI:

```csharp
// MyApp.Shared — interface
public interface IDeviceService
{
    string Platform { get; }
    Task<bool> RequestPermissionAsync(string permission);
}

// MyApp.MAUI — implementation
public class MauiDeviceService : IDeviceService
{
    public string Platform => DeviceInfo.Platform.ToString();

    public async Task<bool> RequestPermissionAsync(string permission)
    {
        var status = await Permissions.RequestAsync<Permissions.StorageRead>();
        return status == PermissionStatus.Granted;
    }
}

// MyApp.Web — stub/alternative implementation
public class WebDeviceService : IDeviceService
{
    public string Platform => "Web";
    public Task<bool> RequestPermissionAsync(string permission) => Task.FromResult(true);
}
```

## Render Mode Notes for Hybrid

- MAUI Hybrid runs **InteractiveAuto** by default (in-process, no server/WASM needed)
- Do NOT use `@rendermode InteractiveServer` on components in MAUI context
- Web project can use `@rendermode InteractiveWebAssembly` or `InteractiveServer`
- Use `#if` or DI abstractions to handle MAUI vs Web differences

## JS Interop in MAUI Hybrid

```csharp
@inject IJSRuntime JS

private async Task CallNativeApi()
{
    // Works in MAUI Hybrid — JS runs in the native WebView
    await JS.InvokeVoidAsync("window.myApp.showNativeAlert", "Hello from Blazor!");
}
```

## EFCore in MAUI Hybrid (.NET 10)

Use SQLite for local persistence:

```csharp
// MauiProgram.cs
var dbPath = Path.Combine(FileSystem.AppDataDirectory, "myapp.db");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite($"Data Source={dbPath}"));
```

## Common Pitfalls

- `FileSystem.AppDataDirectory` must be used for SQLite path (not relative paths)
- CSS isolation works — `Component.razor.css` scoped styles are supported
- Hot Reload works on Windows via `blazor.webview.js` dev tools in DEBUG
- MAUI Hybrid does NOT support `NavigationManager.NavigateTo` with external URLs — use `Launcher.OpenAsync()`
- Avoid `HttpClient` in MAUI unless using `IHttpClientFactory` with platform handlers

## .csproj — MAUI Hybrid (.NET 10)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net10.0-android;net10.0-ios;net10.0-maccatalyst;net10.0-windows10.0.19041.0</TargetFrameworks>
    <OutputType>Exe</OutputType>
    <UseMaui>true</UseMaui>
    <SingleProject>true</SingleProject>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Maui.Controls" Version="$(MauiVersion)" />
    <PackageReference Include="Microsoft.AspNetCore.Components.WebView.Maui" Version="$(MauiVersion)" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApp.Shared\MyApp.Shared.csproj" />
    <ProjectReference Include="..\MyApp.Core\MyApp.Core.csproj" />
  </ItemGroup>
</Project>
```