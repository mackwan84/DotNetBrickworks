# MassTransit 入门指南

MassTransit 是一个用于 .NET 的开源分布式应用程序框架。它在支持的消息传输之上提供了一个消息抽象层。MassTransit 让你专注于增加业务价值，而不是担心消息传递的复杂性。

MassTransit 支持多种消息传输技术。以下是几种流行的技术：

- RabbitMQ
- Azure Service Bus
- Amazon SQS
- Kafka

今天，我将向你展示如何在.NET 中安装和配置 MassTransit。我们将把 MassTransit 连接到几个消息代理（RabbitMQ 和 Azure Service Bus）。同时，我们还会介绍如何使用 MassTransit 发布和消费消息。

## 为什么使用 MassTransit？

MassTransit 解决了构建分布式应用的许多挑战。你（几乎）不需要考虑底层的消息传输。这让你可以专注于提供业务价值。

以下是 MassTransit 为你做的几件事：

- **消息路由** - 基于类型的发布/订阅，自动代理拓扑配置
- **异常处理** - 消息可以重试或移动到错误队列
- **依赖注入** - 服务集合配置和作用域服务提供者
- **请求/响应** - 使用自动响应路由处理请求
- **可观测性** - 原生 Open Telemetry (OTEL) 支持
- **调度** - 使用传输延迟、Quartz.NET 或 Hangfire 安排消息传递
- **Sagas** - 可靠、持久、事件驱动的流程编排

接下来让我们看看如何开始使用 MassTransit。

## 安装和配置 MassTransit 与 RabbitMQ

你需要安装 MassTransit 库。如果你已经有一个消息传输工具，可以安装相应的传输库。让我们添加 MassTransit.RabbitMQ 库来配置 RabbitMQ 作为传输机制。

```bash
Install-Package MassTransit
Install-Package MassTransit.RabbitMQ
```

然后，你可以配置 MassTransit 所需的服务。 AddMassTransit 方法接受一个委托，你可以在其中配置许多设置。例如，你可以通过调用 SetKebabCaseEndpointNameFormatter 来设置消息端点的命名法。这也是你配置传输机制的地方。调用 UsingRabbitMq 允许你将 RabbitMQ 作为传输机制进行连接。

```javascript
builder.Services.AddMassTransit(busConfigurator =>
{
    busConfigurator.SetKebabCaseEndpointNameFormatter();

    busConfigurator.UsingRabbitMq((context, configurator) =>
    {
        configurator.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        configurator.ConfigureEndpoints(context);
    });
});
```

MassTransit 负责设置所需的代理拓扑。RabbitMQ 支持交换机和队列，因此消息是发送或发布到交换机的。RabbitMQ 通过交换机将这些消息路由到相应的队列。

接下来你可以在 Docker 容器中本地启动 RabbitMQ：

```bash
docker run -d --name rabbitmq -p 5672:5672
```

## 使用 Azure Service Bus 配置 MassTransit

Azure Service Bus 是一个基于云的消息代理，支持队列和主题。MassTransit 完全支持 Azure Service Bus，包括许多高级功能和特性。但是，你必须使用 Microsoft Azure Service Bus 服务的标准或高级层级。

要配置 MassTransit 以与 Azure Service Bus 协同工作，你需要安装所需的传输库：

```bash
Install-Package MassTransit.Azure.ServiceBus.Core
```

然后，你可以通过调用 UsingAzureServiceBus 并提供连接字符串来连接到 Azure Service Bus。其他一切保持不变。MassTransit 负责配置代理拓扑。MassTransit 将消息发送到主题，Azure Service Bus 将这些消息路由到相应的队列。

```javascript
builder.Services.AddMassTransit(busConfigurator =>
{
    busConfigurator.SetKebabCaseEndpointNameFormatter();

    busConfigurator.UsingAzureServiceBus((context, configurator) =>
    {
        configurator.Host("<CONNECTION_STRING>");

        configurator.ConfigureEndpoints(context);
    });
});
```

## 使用 MassTransit 内存传输

你还可以配置 MassTransit 使用内存传输。这对于测试很有用，因为它不需要运行消息代理。另一个优点是速度快。

然而，内存传输存在一个大问题：**它不是持久的**。

如果消息总线停止，所有消息都会丢失。所以不要在生产系统中使用内存传输。

它只能在单台机器上工作。因此，对于分布式应用程序来说，这没有意义。

```javascript
builder.Services.AddMassTransit(busConfigurator =>
{
    busConfigurator.SetKebabCaseEndpointNameFormatter();

    busConfigurator.UsingInMemory((context, configurator) =>
    {
        configurator.ConfigureEndpoints(context);
    });
});
```

## 消息类型

MassTransit 要求消息类型为引用类型。因此，你可以使用 class 、 record 或 interface 来定义一个消息。

你通常会创建两种类型的消息：命令和事件。

命令是指示执行某些操作的指令。命令旨在只有一个消费者。命令使用动词开头表达：*CreateArticle*，*PublishArticle*，*ShareArticle*。

事件表示发生了某些重要的事情。事件可以有一个或多个消费者。事件的名称应为过去时态：*ArticleCreated*，*ArticlePublished*，*ArticleShared*。

以下是一个包含文章信息的*ArticleCreated*消息示例：

```java
public record ArticleCreated
{
    public Guid Id { get; init; }
    public string Title { get; init; }
    public string Content { get; init; }
    public DateTime CreatedOnUtc { get; init; }
}
```

使用 **public set** 或 **public init** 属性是推荐的做法，以避免与 System.Text.Json 相关的序列化问题。

## 发布和消费消息

你可以使用 **IPublishEndpoint** 服务通过 MassTransit 发布消息。该框架根据消息类型将消息路由到相应的队列或主题。

```javascript
app.MapPost("article", async (
    CreateArticleRequest request,
    IPublishEndpoint publishEndpoint) =>
{
    await publishEndpoint.Publish(new ArticleCreated
    {
        Id = Guid.NewGuid(),
        Title = request.Title,
        Content = request.Content,
        CreatedOnUtc = DateTime.UtcNow
    });

    return Results.Accepted();
});
```

要消费一个 *ArticleCreated* 消息，你需要实现 **IConsumer** 接口。 **IConsumer** 有一个 Consume 方法，你可以在其中放置你的业务逻辑。消费者还为你提供了对 **ConsumeContext** 的访问权限，你可以用它来发送更多消息。

```java
public class ArticleCreatedConsumer(ApplicationDbContext dbContext)
    : IConsumer<ArticleCreatedEvent>
{
    public async Task Consume(ConsumeContext<ArticleCreatedEvent> context)
    {
        ArticleCreated message = context.Message;

        var article = new Article
        {
            Id = message.Id,
            Title = message.Title,
            Content = message.Content,
            CreatedOnUtc = message.CreatedOnUtc
        };

        dbContext.Add(article);

        await dbContext.SaveChangesAsync();
    }
}
```

MassTransit 并不会自动知道 **ArticleCreatedConsumer** 的存在。在调用 AddMassTransit 时，你必须配置消费者。 AddConsumer 方法将消费者类型注册到总线。

```javascript
builder.Services.AddMassTransit(busConfigurator =>
{
    busConfigurator.SetKebabCaseEndpointNameFormatter();
    busConfigurator.AddConsumer<ArticleCreatedConsumer>();
});
```

## 下一步

MassTransit 是一个我在构建分布式应用时经常使用的优秀消息库。设置非常简单，只有几个重要的抽象概念。你需要了解用于发布消息（**IPublishEndpoint**）和消费消息（**IConsumer**）的抽象。MassTransit 会为你处理繁重的工作。

如果你还没有使用它，我强烈建议将 MassTransit 添加到你的工具箱中。

今天就到这里。期待与你再次相见。
