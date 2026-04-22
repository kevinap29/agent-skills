# Blazor Blueprint Skill

## Description

Expert skill for building Blazor UIs using Blazor Blueprint, a shadcn/ui-inspired component library for Blazor with 115+ accessible components, Tailwind CSS, dark mode, and headless primitives. Use this skill whenever the user mentions Blazor Blueprint, BlazorBlueprint.Components, BlazorBlueprint.Primitives, shadcn-style components in Blazor, or asks about Blazor UI with Tailwind theming. Also use when the user wants accessible, modern, or design-system-first Blazor UI components for .NET 8, 9, or 10 projects including MAUI Hybrid and Blazor Web.

## What is Blazor Blueprint?

Blazor Blueprint brings the shadcn/ui design system to Blazor. It ships 115+ enterprise-class components 
built on Tailwind CSS, with zero Node.js/npm required for consumption, built-in dark mode, full keyboard 
accessibility, and a headless primitives layer for complete style control.

Docs: https://blazorblueprintui.com  
GitHub: https://github.com/blazorblueprintui/ui

## Installation

### Option A — Project Template (Recommended for new projects)

```bash
# Install template
dotnet new install BlazorBlueprint.Templates

# Create project (default: Blazor Server)
dotnet new blazorblueprint -n MyApp

# Target .NET 10
dotnet new blazorblueprint -n MyApp -F net10.0

# WebAssembly mode
dotnet new blazorblueprint -n MyApp -R WebAssembly

# Auto (Server then WASM)
dotnet new blazorblueprint -n MyApp -R Auto
```

### Option B — Add to Existing Project

```bash
# Styled components (includes Primitives)
dotnet add package BlazorBlueprint.Components

# Or only headless primitives
dotnet add package BlazorBlueprint.Primitives

# Optional: Lucide icon set
dotnet add package BlazorBlueprint.Icons.Lucide
```

## Setup in App.razor / index.html

```html
<!-- Theme CSS variables (colors, radius, etc.) -->
<link href="css/theme.css" rel="stylesheet" />

<!-- Pre-built compiled Tailwind + component styles (no build step needed) -->
<link href="_content/BlazorBlueprint.Components/blazorblueprint.css" rel="stylesheet" />
```

## _Imports.razor

```razor
@using BlazorBlueprint.Components
@using BlazorBlueprint.Components.Layout
@using BlazorBlueprint.Components.Forms
@using BlazorBlueprint.Components.Feedback
@using BlazorBlueprint.Icons.Lucide
```

## Program.cs — Service Registration

```csharp
builder.Services.AddBlazorBlueprint();

// Optional services (auto-included with AddBlazorBlueprint)
// builder.Services.AddSingleton<IPortalService, PortalService>();
// builder.Services.AddScoped<DialogService>();
// builder.Services.AddScoped<ToastService>();
```

## Component Examples

### Button

```razor
<BbButton Variant="ButtonVariant.Default" OnClick="HandleClick">
    Save Changes
</BbButton>

<BbButton Variant="ButtonVariant.Destructive" Size="ButtonSize.Sm">
    Delete
</BbButton>

<BbButton Variant="ButtonVariant.Outline" Disabled="@isLoading">
    @if (isLoading) { <BbSpinner Size="SpinnerSize.Sm" /> } else { <span>Submit</span> }
</BbButton>
```

### Card

```razor
<BbCard>
    <BbCardHeader>
        <BbCardTitle>User Profile</BbCardTitle>
        <BbCardDescription>Manage your account settings</BbCardDescription>
    </BbCardHeader>
    <BbCardContent>
        <p>Card content here</p>
    </BbCardContent>
    <BbCardFooter>
        <BbButton>Save</BbButton>
    </BbCardFooter>
</BbCard>
```

### Dialog (Programmatic via DialogService)

```csharp
@inject DialogService DialogService

private async Task OpenDialog()
{
    var result = await DialogService.ShowAsync<ConfirmDialog>(
        title: "Confirm Action",
        parameters: new() { ["Message"] = "Are you sure?" }
    );
}
```

### Toast Notifications

```csharp
@inject ToastService ToastService

ToastService.ShowSuccess("Saved successfully!");
ToastService.ShowError("Something went wrong.");
ToastService.Show("Custom message", variant: ToastVariant.Warning);
```

### Form with Validation

```razor
<EditForm Model="@model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    
    <BbFormField>
        <BbLabel For="name">Full Name</BbLabel>
        <BbInput @bind-Value="model.Name" Placeholder="Enter your name" />
        <ValidationMessage For="() => model.Name" />
    </BbFormField>

    <BbFormField>
        <BbLabel For="email">Email</BbLabel>
        <BbInput Type="email" @bind-Value="model.Email" />
        <ValidationMessage For="() => model.Email" />
    </BbFormField>

    <BbButton Type="submit">Submit</BbButton>
</EditForm>
```

### Data Table

```razor
<BbTable Items="@users" TItem="UserDto">
    <BbTableColumn Title="Name" Property="u => u.Name" Sortable="true" />
    <BbTableColumn Title="Email" Property="u => u.Email" />
    <BbTableColumn Title="Role">
        <CellTemplate Context="user">
            <BbBadge Variant="BadgeVariant.Secondary">@user.Role</BbBadge>
        </CellTemplate>
    </BbTableColumn>
    <BbTableColumn Title="Actions">
        <CellTemplate Context="user">
            <BbButton Size="ButtonSize.Sm" Variant="ButtonVariant.Ghost"
                      OnClick="() => EditUser(user)">
                Edit
            </BbButton>
        </CellTemplate>
    </BbTableColumn>
</BbTable>
```

### Select / Combobox

```razor
<BbSelect @bind-Value="selectedRole" Placeholder="Select a role">
    <BbSelectItem Value="@("admin")">Administrator</BbSelectItem>
    <BbSelectItem Value="@("user")">Standard User</BbSelectItem>
    <BbSelectItem Value="@("viewer")">Viewer</BbSelectItem>
</BbSelect>
```

## Theming (theme.css)

```css
:root {
    --bb-background: 0 0% 100%;
    --bb-foreground: 222.2 84% 4.9%;
    --bb-primary: 222.2 47.4% 11.2%;
    --bb-primary-foreground: 210 40% 98%;
    --bb-secondary: 210 40% 96%;
    --bb-destructive: 0 84.2% 60.2%;
    --bb-radius: 0.5rem;
}

.dark {
    --bb-background: 222.2 84% 4.9%;
    --bb-foreground: 210 40% 98%;
    --bb-primary: 210 40% 98%;
    --bb-primary-foreground: 222.2 47.4% 11.2%;
}
```

Use themes from https://tweakcn.com or shadcn/ui theme playground directly.

## Dark Mode Toggle

```razor
@inject IJSRuntime JS

<BbButton Variant="ButtonVariant.Ghost" OnClick="ToggleDark">
    <LucideMoon />
</BbButton>

@code {
    private async Task ToggleDark()
    {
        await JS.InvokeVoidAsync("document.documentElement.classList.toggle", "dark");
    }
}
```

## Two-Layer Architecture

| Layer | Package | Use When |
|-------|---------|----------|
| Pre-styled Components | `BlazorBlueprint.Components` | You want shadcn look out of the box |
| Headless Primitives | `BlazorBlueprint.Primitives` | You need full CSS control |

Primitives include: `BbDialogPrimitive`, `BbPopoverPrimitive`, `BbTooltipPrimitive`, etc. — with 
ARIA attributes, focus trapping, keyboard nav, all unstyled.

## Key Services

| Service | Purpose |
|---------|---------|
| `IPortalService` | Render overlays outside component tree |
| `IFocusManager` | Trap/restore focus for modals |
| `IPositioningService` | Position floating elements (dropdowns, tooltips) |
| `IKeyboardShortcutService` | Register global keyboard shortcuts |
| `DialogService` | Programmatically open/close dialogs |
| `ToastService` | Show toast notifications |
| `IBbLocaliz*` | Localization support |

## MCP / AI Integration

Blazor Blueprint ships with a built-in MCP server and `llms.txt` — Claude, Cursor, Copilot, and Windsurf 
auto-generate correct component syntax from first prompt.

## MAUI Hybrid Notes

- Add stylesheet links in `wwwroot/index.html` (not App.razor)
- `DialogService` and `ToastService` work identically in MAUI Hybrid
- Tailwind utilities are available if you download the Tailwind standalone CLI (no Node needed)
- Dark mode via CSS class toggle works natively in `BlazorWebView`