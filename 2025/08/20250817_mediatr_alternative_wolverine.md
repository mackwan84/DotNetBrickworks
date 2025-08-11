# [MediatR 替代方案 - Wolverine](https://medium.com/@thecodeman/mediatr-alternative-wolverine-552f65f6dfd9)

## MediatR 即将商业化

作为 .NET 开发者，我们长期以来一直依赖像 MediatR 这样的库来帮助我们实现中介者模式。它主要用于实现命令查询职责分离（CQRS）模式——将读取与写入分离，并保持我们的代码整洁和可维护。（了解如何在不使用 MediatR 的情况下实现它）。

但是，一场转变正在发生。

MediatR 将很快需要对某些使用场景采用商业许可证。

这引发了对替代方案的兴趣，其中一个名字脱颖而出：Wolverine。

Wolverine 是一个用于 CQRS、消息传递和后台处理的高性能 .NET 库。它使用源生成器构建，替代了 MediatR 并添加了重试、调度、Saga 和 Outbox 支持等高级功能。

Wolverine 不仅仅是 MediatR 的直接替代品。它提供：

- 通过源生成器实现更快的性能
- 内置消息总线，支持本地或分布式消息传递
- 调度、重试和持久化发件箱支持
- 原生 Saga/工作流编排
- 与最小 API 无缝集成 在本文中，我们将使用 Wolverine 构建一个完整的用户 CRUD API，以展示你的应用程序架构可以多么简洁和可扩展。

闲话少说，立即开干！

## 我们正在构建的内容

我们将构建一个最小化 API 来：

- 创建新用户
- 通过 ID 读取用户
- 更新现有用户
- 删除用户

每个操作将使用命令或查询对象实现，并由 Wolverine 处理，保持所有内容的解耦、可测试性和简洁性。

### 第一步：定义用户模型

```csharp
public record User(Guid Id, string Name, string Email);
```

为了模拟持久化，我们定义一个内存存储：

```csharp
public static class InMemoryUsers
{
    public static readonly List<User> Users = new();
}
```

### 第二步：创建命令和查询

每个操作都有其自己的请求模型：

```csharp
public record CreateUser(string Name, string Email);
public record GetUser(Guid Id);
public record UpdateUser(Guid Id, string Name, string Email);
public record DeleteUser(Guid Id);
```

### 第三步：实现处理器

Wolverine 会自动发现你的处理程序类，并根据方法名称和参数类型来路由消息。

#### CreateUserHandler

```csharp
public class CreateUserHandler
{
    public User Handle(CreateUser command)
    {
        var user = new User(Guid.NewGuid(), command.Name, command.Email);
        InMemoryUsers.Users.Add(user);
        return user;
    }
}
```

### GetUserHandler

```csharp
public class GetUserHandler
{
    public User? Handle(GetUser query)
    {
        return InMemoryUsers.Users.FirstOrDefault(u => u.Id == query.Id);
    }
}
```

#### UpdateUserHandler

```csharp
public class UpdateUserHandler
{
    public User? Handle(UpdateUser command)
    {
        var user = InMemoryUsers.Users.FirstOrDefault(u => u.Id == command.Id);
        if (user is null) return null;

        var updated = user with { Name = command.Name, Email = command.Email };
        InMemoryUsers.Users.Remove(user);
        InMemoryUsers.Users.Add(updated);
        return updated;
    }
}
```

#### DeleteUserHandler

```csharp
public class DeleteUserHandler
{
    public bool Handle(DeleteUser command)
    {
        var user = InMemoryUsers.Users.FirstOrDefault(u => u.Id == command.Id);
        if (user is null) return false;

        InMemoryUsers.Users.Remove(user);
        return true;
    }
}
```

### 第四步：使用 Wolverine 配置 Minimal API

- 自动注册消息处理器
Wolverine 会扫描你的程序集，查找任何包含与消息类型（如 CreateUser、UpdateUser 等）匹配的 Handle 或 HandleAsync 方法的类，并注册它们。
- 将 IMessageBus 添加到 DI
这使得 IMessageBus 可在你的整个应用程序中使用——包括在最小 API 端点、服务、后台工作程序等中。
- 启用 Wolverine 中间件和扩展
Wolverine 集成到应用程序管道中，并可选地启用：— 重试策略 -消息调度 -发件箱/持久消息传递 -后台作业
- 启动内部运行时引擎
Wolverine 会根据配置启动一个内部消息引擎来管理本地和外部消息路由（例如 RabbitMQ、Kafka）。

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthorization();
builder.Host.UseWolverine(); // 启用 Wolverine

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();

// 创建
app.MapPost("/users", async (CreateUser request, IMessageBus bus) =>
{
    var user = await bus.InvokeAsync<User>(request);
    return Results.Created($"/users/{user.Id}", user);
});

// 获取
app.MapGet("/users/{id:guid}", async (Guid id, IMessageBus bus) =>
{
    var user = await bus.InvokeAsync<User?>(new GetUser(id));
    return user is null ? Results.NotFound() : Results.Ok(user);
});

// 更新
app.MapPut("/users/{id:guid}", async (Guid id, UpdateUser command, IMessageBus bus) =>
{
    if (id != command.Id) return Results.BadRequest();

    var updated = await bus.InvokeAsync<User?>(command);
    return updated is null ? Results.NotFound() : Results.Ok(updated);
});

// 删除
app.MapDelete("/users/{id:guid}", async (Guid id, IMessageBus bus) =>
{
    var deleted = await bus.InvokeAsync<bool>(new DeleteUser(id));
    return deleted ? Results.NoContent() : Results.NotFound();
});

app.Run();
```

> await bus.InvokeAsync(request);
>
> - request 是一个命令（例如 CreateUser）
> - InvokeAsync(...) 的意思是："将此命令发送给 Wolverine 并返回一个 TResponse 类型的结果——在本例中，是一个 User"

调用后会发生什么？

1. Wolverine 接收 CreateUser 命令。
2. 它找到具有此签名的适当处理程序方法：public User Handle(CreateUser command)
3. 它调用该方法。
4. 它将结果作为 User 返回给你。

注意：使用 Wolverine 时，你不需要实现任何像 IRequest 或 IRequestHandler\<TRequest, TResponse\>这样的接口来定义你的消息或处理器。

## 意外之喜：Wolverine 不仅仅是 MediatR

以下是 Wolverine 除了 MediatR 之外还提供给你的功能：

- **内置消息系统**支持：
  - 内存消息传递（如 MediatR）
  - 使用 RabbitMQ、Azure Service Bus、Kafka 等进行分布式消息传递
- **后台任务与调度** — 你可以安排消息稍后运行。
- **持久化发件箱模式** — 凭借内置的持久化支持，Wolverine 能够安全地将外发消息排队以确保可靠交付 — 无需手动实现发件箱模式。
- **Saga与工作流** — 通过内置的Saga支持，轻松编排多步骤、长时间运行的工作流。

## 总结

如果你正在构建干净的 API、基于 CQRS 的服务或微服务，Wolverine 提供了一个强大的 MediatR 替代方案。它使你的代码保持精简和可测试性，同时解锁以下功能：

- 分布式消息传递
- 调度和重试
- Saga 编排
- 源代码生成实现的高性能

既然 MediatR 正在转向商业模型，Wolverine 对于全新项目和现代架构来说就成为一个更具吸引力的选择。

今天就到这里。期待与你再次相见。
