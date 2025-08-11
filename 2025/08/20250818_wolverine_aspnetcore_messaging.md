# [.NET 中使用 Wolverine 进行消息传递](https://www.nikolatech.net/blogs/wolverine-aspnetcore-messaging)

通信在现代应用程序中至关重要，而 .NET 提供了广泛的库可供使用。

通常，消息传递库专注于进程内或分布式消息传递。MediatR 是进程内消息传递的热门选择，而 MassTransit 则广泛用于分布式消息传递。

然而，一个有趣的替代方案是 Wolverine 库，它支持进程内和分布式消息传递。

## Wolverine

Wolverine 是一个用于在 .NET 应用程序中处理消息的强大库。根据你的需求，它可以作为一个简单的中介器或作为一个功能齐全的异步消息框架。

此外，借助 WolverineFx.Http 库，Wolverine 管道可作为 ASP.NET Core 终端提供程序的替代方案。

Wolverite 的突出特点之一是其对事务性发件箱的内置支持，即使是内存中的场景也不例外。

有趣的是，对于横切关注点，Wolverine 使用了直接集成到消息处理程序中的中间件，从而产生了更高效的运行时管道。

闲话不多说，让我们直接开始吧！

## 入门指南

要开始使用 Wolverine，你首先需要安装必要的 NuGet 包。你可以通过 NuGet 包管理器执行此操作，或者在包管理器控制台中运行以下命令：

```bash
dotnet add package WolverineFx
```

根据你的消息传输方式，你可能需要安装额外的库，并且仅需对配置进行微调。

## 规则与约定

Wolverine 利用命名约定来简化其设置和配置。一旦你习惯了这种方法，它会减少样板代码并简化开发流程。

遵循命名约定的处理程序可确保一致性。命名约定：

- 处理程序类型名称应以 **Handler** 或 **Consumer** 结尾。
- 处理程序方法名称应为 **Handle()** 或 **Consume()**。

```csharp
public record CreateProductCommand(string Name, string Description, decimal Price);

public class CreateProductCommandHandler(IMessageBus bus, IApplicationDbContext dbContext)
{
    public async Task<Result<Guid>> Handle(
        CreateProductCommand request,
        CancellationToken cancellationToken)
    {
        var product = new Product(
            Guid.NewGuid(),
            DateTime.UtcNow,
            request.Name,
            request.Description,
            request.Price);

        dbContext.Products.Add(product);

        await dbContext.SaveChangesAsync(cancellationToken);

        var message = new ProductCreated(product.Id, product.Name, product.Description);
        await bus.PublishAsync(message);

        return Result.Success(product.Id);
    }
}
```

消息和消息处理程序必须是具有公共构造函数的公共类型。这是由于 Wolverine 的代码生成策略所要求的。

handler 方法的第一个参数必须是消息类型。

Wolverine 假设处理程序方法的第一个参数是消息类型，而其他参数则从底层 IoC 容器中推断为服务。这种方法注入支持有助于最小化通常所需的样板代码。

```csharp
public class UpdateProductCommandHandler
{
    public async Task<Result> Handle(
        UpdateProductCommand request,
        IApplicationDbContext dbContext,
        CancellationToken cancellationToken)
    {
        var product = await dbContext.Products
            .FirstOrDefaultAsync(x => x.Id == request.Id, cancellationToken);

        if (product is null)
        {
            return Result.NotFound();
        }

        product.Update(request.Name, request.Description, request.Price);

        await dbContext.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

## 内存配置

内存配置再简单不过了，在 Program.cs 中，你只需要写一行代码来注册 wolverine：

```csharp
builder.Host.UseWolverine();
```

这会将 Wolverine 添加到你应用程序的主机中，以最少的设置启用消息处理和命令执行。对于内存传输，这个基本配置将帮助你入门。

## 内存消息传递

Wolverine 的内存消息传递是在同一应用程序内部进行轻量级、快速且高效的通信。

**IMessageBus** 是其消息基础设施的核心接口。它使应用程序的各个部分之间或分布式系统中的服务之间能够进行通信。

**IMessageBus** 的关键方法之一是 **InvokeAsync**，我使用它来调用命令并从处理程序获取结果

```csharp
app.MapPost("products", async (
    CreateProductRequest request,
    IMessageBus bus,
    CancellationToken cancellationToken) =>
{
    var command = request.Adapt<CreateProductCommand>();

    var response = await bus.InvokeAsync<Result<Guid>>(command, cancellationToken);

    return response.IsSuccess
        ? Results.Ok(response.Value)
        : Results.BadRequest();
}).WithTags(Tags.Products);
```

## RabbitMQ 配置

接下来，我将使用 RabbitMQ 作为我的消息传输工具。

> 注意：你也可以轻松配置其他传输方式，Wolverine 支持多种选项，包括 Amazon SQS、Azure Service Bus、Kafka 等。

```csharp
builder.Host.UseWolverine(options =>
{
    options.UseRabbitMq(new Uri("amqp://rabbitmq:5672"))
        .AutoProvision();
});
```

通过使用 **UseRabbitMq** 方法，你可以指定 RabbitMQ 的地址和端口。

**AutoProvision** 选项确保所需的交换机和队列自动创建。没有此选项，你将需要手动设置所有必要的组件。

## 分布式消息传递

如前所述，IMessageBus 负责发送和接收消息。现在唯一的区别是，你需要为想要使用传输发布的消息添加额外配置，如下所示：

```csharp
builder.Host.UseWolverine(options =>
{
    options.UseRabbitMq(new Uri("amqp://rabbitmq:5672"))
        .AutoProvision();

    options.PublishMessage<ProductCreated>().ToRabbitQueue("product-queue");
});
```

或者，你可以使用 **PublishAllMessages** 方法自动注册所有消息：

```csharp
builder.Host.UseWolverine(options =>
{
    options.UseRabbitMq(new Uri("amqp://rabbitmq:5672"))
        .AutoProvision();

    options.PublishAllMessages().ToRabbitQueue("product-queue");
});
```

一旦消息被发布，Wolverine 的消息总线将会将其路由到相应的消费者：

```csharp
public class ProductCreatedConsumer
{
    public void Consume(ProductCreated message, ILogger<ProductCreatedConsumer> logger)
    {
        logger.LogInformation("Message received successfully: {Id} {Name} {Description}",
            message.Id,
            message.Name,
            message.Description);
    }
}
```

此外，Wolverine 内置了对消息处理失败时重试的支持。在可能发生网络或服务故障的分布式系统中，这一点尤为重要。

## 发件箱 & 收件箱模式

Wolverine 可以通过事务性收件箱和发件箱模式与多种数据库引擎和持久化工具集成，以实现持久化消息传递。

要使 Wolverine 的外箱功能将消息持久化存储在持久消息存储中，你需要显式地将传出订阅者端点配置为持久的。或者，你也可以使用内置策略将它们全局设置为持久的：

```csharp
builder.Host.UseWolverine(options =>
{
    options.PublishAllMessages().ToPort(5555).UseDurableOutbox();

    // 强制所有对外订阅者使用持久化消息
    options.Policies.UseDurableOutboxOnAllSendingEndpoints();
});
```

要将单个或所有侦听端点注册到 Wolverine 收件箱机制中，请使用以下选项之一：

```csharp
builder.Host.UseWolverine(options =>
{
    options.ListenAtPort(5555).UseDurableInbox();

    // 强制所有侦听端点使用持久化存储
    options.Policies.UseDurableInboxOnAllListeners();
});
```

## 总结

Wolverine 是一个有趣且强大的库，它弥合了 .NET 应用程序中进程内消息传递和分布式消息传递之间的差距。

其无缝集成、内置的事务性发件箱使其成为 MediatR 和 MassTransit 的有力替代方案。

通过利用约定，Wolverine 简化了设置并减少了样板代码，使我们能够专注于构建可扩展和可维护的应用程序。

无论你是在寻找高效的中间件还是功能完备的消息传递框架，Wolverine 都提供了一个灵活且性能优化的解决方案，值得探索。

今天就到这里。期待与你再次相见。
