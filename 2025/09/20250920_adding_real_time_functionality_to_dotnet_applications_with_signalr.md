# [使用 SignalR 为 .NET 应用程序添加实时功能](https://www.milanjovanovic.tech/blog/adding-real-time-functionality-to-dotnet-applications-with-signalr)

当今的现代应用程序必须在不刷新用户界面的情况下提供最新信息。

如果你需要在 .NET 应用程序中引入实时功能，有一个库你很可能会选择——SignalR。

SignalR 允许你将服务器端代码的内容实时推送到任何连接的客户端，因为变化是实时发生的。

以下是我今天将要与你探讨的内容：

- 创建你的第一个 SignalR Hub
- 使用 Postman 测试 SignalR
- 创建强类型的 Hub
- 向特定用户发送消息

让我们看看 SignalR 为何如此强大，以及使用它构建实时应用程序是多么容易。

## 安装和配置 SignalR

要开始使用 SignalR，你需要：

- 安装 NuGet 包
- 创建 Hub 类
- 注册 SignalR 服务
- 映射并公开集线器端点，以便客户端可以连接到它

让我们首先安装 Microsoft.AspNetCore.SignalR.Client NuGet 包：

```bash
Install-Package Microsoft.AspNetCore.SignalR.Client
```

然后你需要一个 SignalR Hub ，它是应用程序中负责管理客户端和发送消息的核心组件。

让我们通过继承基 Hub 类来创建一个 NotificationsHub ：

```csharp
public sealed class NotificationsHub : Hub
{
    public async Task SendNotification(string content)
    {
        await Clients.All.SendAsync("ReceiveNotification", content);
    }
}
```

SignalR Hub 暴露了一些有用的属性：

- Clients - 用于调用连接到此集线器的客户端上的方法
- Groups - 用于从组中添加和移除连接的抽象
- Context - 用于访问有关中心调用者连接的信息

最后，你需要通过调用 AddSignalR 方法来注册 SignalR 服务。你还需要调用 MapHub\<T\> 方法，在该方法中指定 NotificationsHub 类以及客户端将用于连接到中心的路径。

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR();

var app = builder.Build();

app.MapHub<NotificationsHub>("notifications-hub");

app.Run();
```

现在让我们看看如何测试 NotificationsHub 。

## 从 Postman 连接到 SignalR Hub

要测试 SignalR，你需要一个客户端来连接到 Hub 实例。你可以使用 Blazor 或 JavaScript 创建一个简单的应用程序，但我将向你展示一种不同的方法。

我们将使用 Postman 的 WebSocket 请求连接到 NotificationsHub 。

![Postman WebSocket 请求](hhttps://www.milanjovanovic.tech/blogs/mnw_043/postman_websocket_request.png?imwidth=2048)

以下是我们需要做的：

- 连接到 NotificationsHub
- 将通信协议设置为 JSON
- 发送消息以调用 NotificationsHub 方法

所有消息都需要以空终止符结尾，它只是 ASCII 字符 0x1E 。

让我们先发送此消息以将通信协议设置为 JSON：

```json
{
  "protocol": "json",
  "version": 1
}
```

你将从集线器收到此响应。

![Postman 设置协议请求](https://www.milanjovanovic.tech/blogs/mnw_043/postman_set_protocol_request.png?imwidth=3840)

我们需要稍微不同的消息格式来调用 Hub 上的消息。关键是指定 arguments 和 target ，这是我们想要调用的实际中心方法。

假设我们想要调用 NotificationsHub 上的 SendNotification 方法：

```json
{
  "arguments": ["This is the notification message."],
  "target": "SendNotification",
  "type": 1
}
```

这将是我们从 NotificationsHub 获得的响应：

![Postman 发送通知请求](https://www.milanjovanovic.tech/blogs/mnw_043/postman_send_notification_request.png?imwidth=3840)

## 创建强类型的 Hub

基 Hub 类使用 SendAsync 方法向连接的客户端发送消息。不幸的是，我们必须使用字符串来指定要调用的客户端方法，这很容易出错。而且也没有任何机制来强制使用哪些参数。

SignalR 支持强类型中心，旨在解决这个问题。

首先，你需要定义一个客户端接口，让我们创建一个简单的 INotificationsClient 抽象：

```csharp
public interface INotificationsClient
{
    Task ReceiveNotification(string content);
}
```

参数不一定是基本类型，也可以是对象。SignalR 将处理客户端的序列化工作。

之后，你需要更新 NotificationsHub 类，使其继承自 Hub\<T\> 类，以使其成为强类型：

```csharp
public sealed class NotificationsHub : Hub<INotificationsClient>
{
    public async Task SendNotification(string content)
    {
        await Clients.All.ReceiveNotification(content);
    }
}
```

你将失去对 SendAsync 方法的访问权限，只有客户端接口中定义的方法才可用。

## 使用 HubContext 发送服务器端消息

如果我们无法从后端向连接的客户端发送通知，那 NotificationsHub 还有什么用呢？

确实不多。

你可以在后端代码中使用 IHubContext\<THub\> 接口访问 Hub 实例。

你可以使用 IHubContext\<THub, TClient\> 来实现强类型 Hub。

这是一个简单的最小 API 端点，它为我们的强类型 Hub 注入了一个 IHubContext\<NotificationsHub, INotificationsClient\> ，并使用它向所有连接的客户端发送通知：

```csharp
app.MapPost("notifications/all", async (
    string content,
    IHubContext<NotificationsHub, INotificationsClient> context) =>
{
    await context.Clients.All.ReceiveNotification(content);

    return Results.NoContent();
});
```

## 向特定用户发送消息

SignalR 的真正价值在于能够向特定用户发送消息，或在本例中发送通知。

我见过一些复杂的实现，它们管理着一个包含用户标识符和活动连接映射的字典。当 SignalR 已经支持此功能时，为什么还要这样做呢？

你可以调用 User 方法并传入 userId ，将 ReceiveNotification 消息限定到该特定用户。

```csharp
app.MapPost("notifications/user", async (
    string userId,
    string content,
    IHubContext<NotificationsHub, INotificationsClient> context) =>
{
    await context.Clients.User(userId).ReceiveNotification(content);

    return Results.NoContent();
});
```

SignalR 如何知道要将消息发送给哪个用户？

它在内部使用 DefaultUserIdProvider 从声明中提取用户标识符。具体来说，它使用的是 ClaimTypes.NameIdentifier 声明。这也意味着在连接到 Hub 时你应该已经通过身份验证，例如通过传递 JWT。

```csharp
public class DefaultUserIdProvider : IUserIdProvider
{
    public virtual string? GetUserId(HubConnectionContext connection)
    {
        return connection.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    }
}
```

默认情况下，中心中的所有方法都可以由未经身份验证的用户调用。因此，你需要使用 Authorize 属性对其进行装饰，以仅允许经过身份验证的客户端访问该中心。

```csharp
[Authorize]
public sealed class NotificationsHub : Hub<INotificationsClient>
{
    public async Task SendNotification(string content)
    {
        await Clients.All.ReceiveNotification(content);
    }
}
```

## 总结

为你的应用程序添加实时功能为创新创造了空间，并为用户增加了价值。

使用 SignalR，你可以在几分钟内开始在 .NET 中构建实时应用程序。

你需要掌握一个概念—— Hub 类。SignalR 抽象化了消息传输机制，因此你不必担心它。

确保向 SignalR 中心发送经过身份验证的请求，并在 Hub 上启用身份验证。SignalR 将在内部跟踪连接到你的中心的用户，允许你根据用户标识符向他们发送消息。

今天就到这里。期待与你再次相见。
