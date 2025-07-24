# 使用 MassTransit 的请求-响应消息模式

构建分布式应用乍一看可能很简单。不过是服务器之间相互通信嘛。对吧？

然而，它带来了一系列你必须考虑的潜在问题。如果网络出现故障怎么办？服务意外崩溃怎么办？你尝试扩展，结果一切在负载下崩溃怎么办？这就是你的分布式系统通信方式变得至关重要的地方。

传统的同步通信，即服务之间直接相互调用，本质上是非常脆弱的。它造成了紧密耦合，使得整个应用容易受到单点故障的影响。

为了应对这一问题，我们可以转向分布式消息传递（并引入一整套不同的问题，但这又是另一个话题了）。

在 .NET 领域中实现这一目标的一个强大工具是 MassTransit。

今天，我们将探讨 MassTransit 对请求-响应模式的实现。

## 请求-响应消息模式简介

让我们首先解释一下请求-响应消息模式是如何工作的。

请求-响应模式就像进行传统的函数调用，但通过网络实现。一个服务，即请求者，发送一个请求消息并等待相应的响应消息。从请求者的角度来看，这是一种同步通信方式。

### 优点

- **松散耦合**：服务之间不需要直接了解对方，只需了解消息契约。这使得变更和扩展变得更加容易。
- **位置透明性**：请求者无需知道响应者的位置，从而提高了灵活性。

### 缺点

- **延迟**： 消息传递的开销增加了一些额外的延迟。
- **复杂性**：引入消息系统和管理额外的基础设施可能会增加项目复杂性。

## 使用 MassTransit 的请求-响应消息模式

MassTransit 默认支持请求-响应消息模式。我们可以使用请求客户端发送请求并等待响应。请求客户端是异步的，并支持 await 关键字。请求还将默认具有 30 秒的超时时间，以防止等待响应过久。

让我们设想一个场景，你有一个订单处理系统，需要获取订单的最新状态。我们可以从一个订单管理服务中获取状态。使用 MassTransit，你将创建一个请求客户端来启动该过程。这个客户端将发送一个 *GetOrderStatusRequest* 消息到总线。

```java
public record GetOrderStatusRequest
{
    public string OrderId { get; init; }
}
```

在订单管理端，一个响应者（或消费者）将监听 *GetOrderStatusRequest* 消息。它接收请求，可能查询数据库以获取状态，然后向总线发送一个 *GetOrderStatusResponse* 消息。原始请求客户端将等待此响应，并据此进行相应处理。

```java
public class GetOrderStatusRequestConsumer : IConsumer<GetOrderStatusRequest>
{
    public async Task Consume(ConsumeContext<GetOrderStatusRequest> context)
    {
        // 从数据库中获取订单状态

        await context.ResponseAsync<GetOrderStatusResponse>(new
        {
            // 设置响应属性
        });
    }
}
```

## 在模块化单体中获取用户权限

这里有一个真实场景，我的团队决定实施这种模式。我们正在构建一个模块化单体，其中一个模块负责管理用户权限。其他模块可以调用用户模块来获取用户的权限。在我们仍然处于单体系统内部时，这种方式运行得很好。

然而，在某个时刻，我们需要将一个模块提取到一个独立的服务中。这意味着通过简单方法调用与用户模块的通信将不再有效。

幸运的是，我们已经在系统中使用 MassTransit 和 RabbitMQ 进行消息传递。

所以，我们决定使用 MassTransit 的请求-响应功能来实现这一点。

新服务将注入一个 **IRequestClient<GetUserPermissions>** 。我们可以使用它来发送一个 *GetUserPermissions* 消息并等待响应。

MassTransit 的一个非常强大的特性是，你可以等待多个响应消息。在这个例子中，我们正在等待一个 *PermissionsResponse* 或 *Error* 响应。这很棒，因为我们也有一方法在消费者中处理失败情况。

```java
internal sealed class PermissionService(
    IRequestClient<GetUserPermissions> client)
    : IPermissionService
{
    public async Task<Result<PermissionsResponse>> GetUserPermissionsAsync(
        string identityId)
    {
        var request = new GetUserPermissions(identityId);

        Response<PermissionsResponse, Error> response =
            await client.GetResponse<PermissionsResponse, Error>(request);

        if (response.Is(out Response<Error> errorResponse))
        {
            return Result.Failure<PermissionsResponse>(errorResponse.Message);
        }

        if (response.Is(out Response<PermissionsResponse> permissionResponse))
        {
            return permissionResponse.Message;
        }

        return Result.Failure<PermissionsResponse>(NotFound);
    }
}
```

在用户模块中，我们可以轻松实现 *GetUserPermissionsConsumer* 。如果找到权限，它将返回 *PermissionsResponse* ；如果失败，则返回 *Error* 。

```java
public sealed class GetUserPermissionsConsumer(
    IPermissionService permissionService)
    : IConsumer<GetUserPermissions>
{
    public async Task Consume(ConsumeContext<GetUserPermissions> context)
    {
        Result<PermissionsResponse> result =
            await permissionService.GetUserPermissionsAsync(
                context.Message.IdentityId);

        if (result.IsSuccess)
        {
            await context.RespondAsync(result.Value);
        }
        else
        {
            await context.RespondAsync(result.Error);
        }
    }
}
```

## 结束语

通过采用 MassTransit 的消息模式，你正在构建一个更加稳固的基础。你的 .NET 服务现在耦合度更低，这为你提供了灵活性，可以独立地演进它们，并应对那些不可避免的网络故障或服务中断。

请求-响应模式是你消息传递工具库中的强大工具。MassTransit 使其实现变得非常简单，确保请求和响应能够可靠地传递。

我们可以使用请求-响应模式来实现模块化单体中各模块之间的通信。然而，不要将其推向极端，否则你的系统可能会因延迟增加而受到影响。

从小处着手，进行实验，看看消息传递的可靠性和灵活性如何改变你的开发体验。

今天就到这里。期待与你再次相见。
