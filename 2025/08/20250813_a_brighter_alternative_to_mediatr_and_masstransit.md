# [Brighter，一个 MediatR 和 MassTransit 的免费替代方案？](https://mareks-082.medium.com/a-brighter-alternative-to-mediatr-and-masstransit-69d46d10b4b8)

多年来，MassTransit 和 MediatR 一直是领先的开源 .NET 消息传递库。但那个时代即将结束：MassTransit 正在转向商业许可证。V8 很可能是其最后一个完全开源的版本。与此同时，MediatR 也正在推出付费方案。

这一转变让许多开发者开始寻找免费的替代方案。其中，Paramore Brighter 是一个越来越受到关注的替代选择。

## 什么是 Brighter？

Brighter 是一个轻量级的、采用 MIT 许可证的开源库，用于在.NET 中构建消息驱动应用程序。它提供了一种灵活的方式来实现命令责任分离（CRS）模式，允许开发人员将命令与其处理程序分离。Brighter 支持各种消息模式，并与流行的消息代理（如 RabbitMQ、Azure Service Bus 和 Amazon SQS）集成。

## MediatR vs Brighter

MediatR 是 .NET 中用于构建消息驱动应用程序的著名库。它允许开发人员解耦请求/响应交互，并支持命令和查询。让我们看看 Brighter 作为替代品表现如何。

### Brigher vs Darker

对我来说，第一个惊喜是 Brighter 库只涵盖命令和事件，而其对应的 Darker（一个独立的库）则负责查询。这种关注点分离与 MediatR 不同，后者在单个库中处理命令和查询。

尽管 Darker 仅比 Brighter 年轻一岁，但功能集却大不相同。Darker 似乎是一个简单的查询库，其开发活跃度也不如 Brighter。Brighter 在 GitHub 上有 2.2k 个星标，而 Darker 只有 200 个，这表明 Brighter 是一个更成熟、被广泛采用的库。

### Brighter 搭配 Darker 能否取代 MediatR？

如果你了解 MediatR，那么使用 Brighter 会让你感到亲切。它采用了处理命令和事件的类似方法。

MediatR 示例：

```csharp
public record CreateOrderCommand(int ProductId, int Quantity) 
  : IRequest<OrderResult>;

public record OrderResult(Guid OrderId, bool Success);

public class CreateOrderHandler(IOrderService orderService) 
  : IRequestHandler<CreateOrderCommand, OrderResult>
{
  public async Task<OrderResult> Handle(
    CreateOrderCommand request, 
    CancellationToken cancellationToken)
  {
    ..
  }
}

// 使用 MediatR 发送命令
var command = new CreateOrderCommand(productId, quantity);
var result = await sender.Send(command);
```

Brighter 示例：

```csharp
public class CreateOrder : Command
{
  public CreateOrder(Guid id) : base(id)
  {
    Id = id;
  }
  public int ProductId { get; set; }
  public int Quantity { get; set; }
}

public record OrderResult(Guid OrderId, bool Success);

public class CreateOrderHandler() : RequestHandlerAsync<CreateOrder>
{
  public override async Task<CreateOrder> HandleAsync(
    CreateOrder command,
    CancellationToken cancellationToken = default)
  {
    // ... processing
    return await base.HandleAsync(command, cancellationToken);
  }
}

// 使用 Brighter 发送命令
var createOrder = new CreateOrder(Guid.NewGuid())
{
  ProductId = 1,
  Quantity = 1
};
await commandProcessor.SendAsync(createOrder);
```

如你所见，命名约定有些不同，但概念仍然相似。你定义命令和处理程序，Brighter 负责通过管道调度它们。

主要区别在于 Brighter 的请求类比 MediatR 的要复杂一些。它们包含 Id(Guid)和 Span(与遥测相关)属性。还建议使用 Command 作为命令的基类，使用 Event 类作为事件的基类。另一方面，MediatR 的 IRequest 只是一个简单的(且空的)标记接口。

### 查询

在 Darker 中处理查询稍微不那么复杂，更接近 MediatR 的体验。要发送查询，你只需要实现 IQuery\<T\> 接口。

MediatR 示例：

```csharp
public record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDto>;

public record OrderDto(
  Guid OrderId, 
  int ProductId, 
  int Quantity, 
  DateTimeOffset CreatedAt);

public class GetOrderByIdQueryHandler(IOrderService orderService)
  : IRequestHandler<GetOrderByIdQuery, OrderDto>
{
  public async Task<OrderDto> Handle(
    GetOrderByIdQuery request, 
    CancellationToken cancellationToken)
  {
    ...
  }
}
var result = await sender.Send(new CreateOrderCommand(productId, quantity));
```

Darker 示例：

```csharp
public record GetOrderById(int OrderId) : IQuery<OrderDto>;

public record OrderDto(
  Guid OrderId, 
  int ProductId, 
  int Quantity, 
  DateTimeOffset CreatedAt);

public class GetOrderByIdHandler : QueryHandlerAsync<GetOrderById, OrderDto>
{
  public override Task<OrderDto> ExecuteAsync(
     GetOrderById query, 
     CancellationToken cancellationToken = new CancellationToken())
  {
    return Task.FromResult(new OrderDto());
  } 
}
var orderResult = await queryProcessor.ExecuteAsync(new GetOrderById(1));
```

Brighter（搭配 Darker）提供了类似于 MediatR 中的调度功能。但那些额外的功能呢？

### 管道

我是管道概念的忠实拥护者，在 MediatR 中被称为行为。Brighter 的实现与 MediatR 略有不同。

你可以将 Brighter 的管道想象成俄罗斯套娃模型，你可以在其中嵌套管道。而 MediatR 的管道更像是一个按顺序执行的扁平行为列表。

在 Brighter 中定义管道需要比在 MediatR 中编写更多的样板代码。Brighter 中最简单的管道如下所示：

```csharp
public class RequestLoggingHandler<TRequest>(
  ILogger<RequestLoggingHandler<TRequest>> logger)
  : RequestHandlerAsync<TRequest> where TRequest : class, IRequest
{
  public override Task<TRequest> HandleAsync(
     TRequest command, 
     CancellationToken cancellationToken = new())
  {
    LogCommand(command);
    return base.HandleAsync(command, cancellationToken);
  }

  private void LogCommand(TRequest request)
  {
    logger.LogInformation("Logging handler pipeline call");
  } 
}

public class RequestLoggingAttribute(int step, HandlerTiming timing) 
  : RequestHandlerAttribute(step, timing)
{
  public override object[] InitializerParams()
  {
    return [Timing];
  }
  public override Type GetHandlerType()
  {
    return typeof(RequestLoggingHandler<>);
  }
}

// 使用自定义处理器
public class CreateOrderHandler() : RequestHandlerAsync<CreateOrder>
{
  [RequestLogging(step: 1, timing: HandlerTiming.Before)]
  public override async Task<CreateOrder> HandleAsync(
     CreateOrder command,
     CancellationToken cancellationToken = default)
  {
    return await base.HandleAsync(command, cancellationToken);
  }
}
```

要使用管道，你需要定义相应的属性并在处理程序中显式使用它。它不像在 MediatR 中那样只是一个类和一个注册。

如果你更倾向于避免基于属性的模式，那么有一个替代方案。基本上，你需要像这样编写代码：

```csharp
var myCommandHandler = new MyCommandHandler();
var myLoggingHandler = new MyLoggingHandler(log);

// 在这里你将所有处理器链接起来
myLoggingHandler.Successor = myCommandHandler;

var subscriberRegistry = new SubscriberRegistry();
subscriberRegistry.Register<MyCommand, MyLoggingHandler>();
```

明确性在某些情况下可能适用，但我更喜欢 MediatR 行为的简单性，而不是这些手动设置。

更成问题的是，Darker 不支持像 Brighter 那样的管道模型。我甚至找不到为查询设置管道的方法。文档也没有帮助，因为 Darker 配置部分的文档是空的。

### 在处理器之间共享数据

Brighter 是一个相当固执己见的库，所以如果他们不提供在处理程序之间共享数据的方式，那会很奇怪。在 Brighter 中，命令调度器自动提供一个上下文包（Context Bag），这是一个简单的 IDictionary\<string, object\> ，它被注入到管道中的每个处理程序中，并在该请求的持续时间内存在。处理程序可以在 Context.Bag 中读取和写入任意数据，而无需触及命令自身的属性。

MediatR 的 IPipelineBehavior\<TRequest,TResponse\> 没有共享上下文，因此要在行为之间共享数据，你需要在 DI 容器中注册一个自定义范围的依赖项（例如字典），并在需要的地方注入它。虽然这种基于 DI 的方法使行为与请求类型保持解耦，但与 Brighter 开箱即用的上下文包相比，它需要额外的设置。

Brighter 上下文包示例：

```csharp
public class CorrelationAttribute(int step, HandlerTiming timing) : RequestHandlerAttribute(step, timing)
{
  public override Type GetHandlerType()
  {
    return typeof(CorrelationHandler<>);
  }
}

public class CorrelationHandler<TRequest>
    : RequestHandlerAsync<TRequest> where TRequest : class, IRequest
{
  public override Task<TRequest> HandleAsync(TRequest command, CancellationToken cancellationToken = new())
  {
    Context.Bag["CorrelationId"] = Guid.NewGuid(); //can be any object
    return base.HandleAsync(command, cancellationToken);
  }
}

[Correlation(step: 1, timing: HandlerTiming.Before)]
public override async Task<CreateOrder> HandleAsync(
  CreateOrder command,
  CancellationToken cancellationToken = default)
{
  var correlationId = Context.Bag["CorrelationId"];
  return await base.HandleAsync(command, cancellationToken);
}
```

### 语义上合理的命令

你可以使用这个方便的上下文从管道中的处理程序传递数据，但另一方面，在处理命令时无法返回任何结果。Brighter 强制执行一个严格的规则，即命令不能返回值。因此，通常的解决方法是让处理程序改变命令的状态：

```csharp
public class CreateOrder : Command
{
  ...
  public bool Success { get; set; }
}

public class CreateOrderHandler() : RequestHandlerAsync<CreateOrder>
{
  public override async Task<CreateOrder> HandleAsync(
    CreateOrder command,
    CancellationToken cancellationToken = default)
  {
    command.ProceededSuccessfully = true; // 修改状态
    return await base.HandleAsync(command, cancellationToken);
  }
}

// 与 MediatR 的 SendAsync 返回 Task<TResponse> 不同，Brighter 的 SendAsync 仅返回非泛型的 Task。
await commandProcessor.SendAsync(createOrder);

if (createOrder.Success)
{
  Console.WriteLine("CreateOrder command processed successfully.");
}
else
{
  Console.WriteLine("CreateOrder command failed.");
}
```

理论上，是的，Brighter 是对的，命令不应该产生结果。然而，我认为这种对输入数据的变更比 MediatR 允许命令返回数据的方法更不可取。

### 小结

虽然 Brighter 可以作为 CRS 库替代 MediatR，但我仍然更喜欢 MediatR 的实用方法，而不是 Brighter 对严格干净消息模式的强调。关于查询部分(Darker)，我不太确定。它能工作，但方式非常有限。

让我们看看 Brighter 在 MassTransit 领域提供什么。

## MassTransit vs Brighter

MassTransit 是一个功能完备的服务总线，提供高级功能，如 sagas、路由单和云代理支持。Brighter 支持其中一些功能，但它不提供与 MassTransit 相同级别的功能。

### 发布/订阅消息传递

当我探索替代方案时，我的首要要求是使用 RabbitMQ 实现简单的发布/订阅消息传递。虽然 Brighter 确实提供发布/订阅支持，但它比 MassTransit 需要更多的配置和设置。我最终让它工作了，但花费了我相当多的时间。

主要问题在于配置，而且文档又一次没有达到预期。此外，实现这种发布/订阅模式迫使你调用两个配置方法： AddBrighter 用于发送消息， AddServiceActivator 用于处理接收消息。我期望它可以在一个方法中完成所有设置。

你最终还需要编写大量样板代码，并为了调度命令而处理不同的类： SendAsync 通过内部总线（如 MediatR），而 PostAsync 则将命令推送到外部总线（当你需要 RabbitMQ 时）。

所有这些让我相信，Brighter 严重受到了抽象泄漏的影响。它给人的印象是，即使要设置最简单的消息模式，你也需要了解很多关于 Brighter 内部工作原理的知识。

### 发件箱 & 收件箱

我寻找的另一种模式是发件箱。在我刚开始职业生涯时，我学到的第一件事就是使用 MS SQL 中的表存储待发送的邮件，然后通过计划任务稍后发送，而不是直接通过 SMTP 发送。这基本上就是发件箱模式，但那时我们并不这样称呼它。

Brighter 内置了对 Outbox 模式的支持。它允许你将需要发送的消息存储在数据库表中，然后稍后处理它们。这对于确保在消息代理发生故障时消息不会丢失非常有用。

然而，我未能成功使 PostgreSQL 的 outbox 与标准异步命令处理一起工作。我在 GitHub 讨论中提出了一个问题，但得到的答复是团队目前正专注于新版本 V10，而 PostgreSQL Outbox 的异步版本目前不受支持。也许在下一个版本中，Brighter 将提供一个更可行的选择。

### 小结

Brighter 确实是 MassTransit 的一个替代方案，但它不仅缺乏像 sagas 和 routing slips 这样的高级功能，还缺少一些基本功能，如异步 outbox 支持。所以我不确定现在是否会推荐它作为 MassTransit 的替代品。

## 总结

Brighter 是 MediatR 或 MassTransit 的可靠替代品吗？根据我的经验，第 9 版本并不完全符合我的需求。学习曲线陡峭，文档也有改进空间。话虽如此，如果你只是想要一个围绕 RabbitMQ 的轻量级代理包装器，Brighter 可以胜任，但我建议还是先等等 v10 版本。

今天就到这里。期待与你再次相见。
