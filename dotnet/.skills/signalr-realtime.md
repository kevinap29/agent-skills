# SignalR & Real-time Skill (.NET 10)

## Description

Expert skill for implementing real-time web functionality using ASP.NET Core SignalR. Covers Hub design, client-side integration (Blazor/JS), broadcasting strategies, groups, and performance optimization. Use this skill when the user needs to add live notifications, chat features, real-time dashboards, or push updates from the server to clients.

## Server-Side Hub

```csharp
public class NotificationHub : Hub
{
    // Called by client
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    // Join a group (e.g., a specific store or order)
    public async Task JoinOrderGroup(string orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, orderId);
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.Identity?.Name;
        // Logic on connection
        await base.OnConnectedAsync();
    }
}
```

## Registration

```csharp
builder.Services.AddSignalR();

// In middleware
app.MapHub<NotificationHub>("/hubs/notifications");
```

## Sending from Outside a Hub (e.g., from a Controller or Handler)

```csharp
public class OrderProcessedHandler : INotificationHandler<OrderProcessedEvent>
{
    private readonly IHubContext<NotificationHub> _hubContext;

    public OrderProcessedHandler(IHubContext<NotificationHub> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task Handle(OrderProcessedEvent notification, CancellationToken ct)
    {
        // Send to a specific group
        await _hubContext.Clients.Group(notification.OrderId.ToString())
            .SendAsync("OrderUpdate", "Your order is ready!", ct);
    }
}
```

## Blazor Client Integration

```razor
@using Microsoft.AspNetCore.SignalR.Client
@inject NavigationManager Navigation
@implements IAsyncDisposable

<div class="alert">@_message</div>

@code {
    private HubConnection? _hubConnection;
    private string? _message;

    protected override async Task OnInitializedAsync()
    {
        _hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/hubs/notifications"))
            .WithAutomaticReconnect()
            .Build();

        _hubConnection.On<string>("OrderUpdate", (msg) =>
        {
            _message = msg;
            InvokeAsync(StateHasChanged);
        });

        await _hubConnection.StartAsync();
    }

    public async ValueTask DisposeAsync()
    {
        if (_hubConnection is not null)
        {
            await _hubConnection.DisposeAsync();
        }
    }
}
```

## SignalR Tips

- **Automatic Reconnect**: Always use `.WithAutomaticReconnect()` on the client to handle network drops.
- **Typed Hubs**: Use `Hub<ITypedHubInterface>` on the server to avoid magic strings when calling `SendAsync`.
- **Scaling**: If running multiple server instances, use **Redis Backplane** to sync messages across servers.
- **Binary Format**: Use **MessagePack** instead of JSON for smaller payloads and better performance in high-frequency apps.
- **Security**: Use `[Authorize]` on Hub classes or methods just like Controllers.
- **Small Payloads**: Keep messages small. Don't send large objects; send IDs or small DTOs and let the client fetch more data if needed.
