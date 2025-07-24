# [.NET 中使用 Redis Pub/Sub 实现简单消息传递](https://www.milanjovanovic.tech/blog/simple-messaging-in-dotnet-with-redis-pubsub)

Redis 是缓存数据的流行选择，但其功能远不止于此。它一个鲜为人知的功能是支持发布/订阅（Pub/Sub）。Redis 通道为在您的 .NET 应用程序中实现实时消息传递提供了一种有趣的方法。然而，如您很快将看到的，通道也有一些缺点。

今天我们将探讨：

- Redis 通道的基础知识
- 通道使用的实际用例
- 在 .NET 中实现一个发布/订阅示例
- 分布式系统中的缓存失效

闲话少说，立即开干！

Redis 通道是命名通信通道，实现了发布/订阅消息传递范式。每个通道由一个唯一的名称标识（例如， **notifications** ， **updates** ）。通道促进了从发布者到订阅者的消息传递。

发布者使用 **PUBLISH** 命令向特定通道发送消息。订阅者使用 **SUBSCRIBE** 命令注册以接收来自通道的消息。

Redis 通道遵循基于主题的发布/订阅模型。多个发布者可以向一个通道发送消息，多个订阅者可以从该通道接收消息。

然而，需要注意的是，Redis 通道不会存储消息。如果在发布消息时没有订阅者，该消息将立即被丢弃。

Redis 通道具有**至多一次传递**的语义。

## 实际用例

鉴于 Redis 通道采用至多一次投递（如果没有订阅者，消息可能会丢失），它们非常适合偶尔消息丢失可接受且需要实时或近实时通信的场景。

以下是几个可能的使用场景：

- **社交媒体动态**：向用户广播新帖子或更新。
- **实时比分更新**：向订阅者发送实时比赛分数或体育更新。
- **聊天应用**：实时向活跃参与者发送聊天消息。
- **协作编辑**：在协作编辑环境中传播更改。
- **分布式缓存更新**：当数据发生变化时，使多个服务器上的缓存条目失效。我们将在本文后面详细讨论这一点。

Redis 通道并不是处理关键数据的最佳选择，因为在这些情况下消息丢失是不可接受的。在这种情况下，您应该考虑使用更可靠的消息系统。

让我们看看如何在 .NET 中使用 Redis 通道。

## 使用 Redis 通道的发布/订阅

我们将使用 **StackExchange.Redis** 库通过 Redis 通道发送消息。

让我们从安装开始：

```bash
Install-Package StackExchange.Redis
```

接下来你可以在 Docker 容器中本地运行 Redis 默认端口是 6379 。

```bash
docker run -it -p 6379:6379 redis
```

这是一个简单的后台服务，它将作为我们的消息 **Producer** 。

我们通过连接到我们的 Redis 实例来创建一个 **ConnectionMultiplexer** 。这使我们能够获得一个 **ISubscriber** ，我们可以用它来进行发布/订阅消息传递。 **ISubscriber** 将允许我们通过指定通道名称向一个通道发布消息。

```java
public class Producer(ILogger<Producer> logger) : BackgroundService
{
    private static readonly string ConnectionString = "localhost:6379";
    private static readonly ConnectionMultiplexer Connection =
        ConnectionMultiplexer.Connect(ConnectionString);

    private const string Channel = "messages";

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var subscriber = Connection.GetSubscriber();

        while (!stoppingToken.IsCancellationRequested)
        {
            var message = new Message(Guid.NewGuid(), DateTime.UtcNow);

            var json = JsonSerializer.Serialize(message);

            await subscriber.PublishAsync(Channel, json);

            logger.LogInformation(
                "Sending message: {Channel} - {@Message}",
                message);

            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

让我们也引入一个单独的后台服务来消费消息。

**Consumer** 连接到同一个 Redis 实例并获取一个 **ISubscriber** 。 **ISubscriber** 提供了一个 **SubscribeAsync** 方法，我们可以使用它来订阅来自特定通道的消息。这个方法接受一个回调委托，我们可以用它来处理消息。

```java
public class Consumer(ILogger<Consumer> logger) : BackgroundService
{
    private static readonly string ConnectionString = "localhost:6379";
    private static readonly ConnectionMultiplexer Connection =
        ConnectionMultiplexer.Connect(ConnectionString);

    private const string Channel = "messages";

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var subscriber = Connection.GetSubscriber();

        await subscriber.SubscribeAsync(Channel, (channel, message) =>
        {
            var message = JsonSerializer.Deserialize<Message>(message);

            logger.LogInformation(
                "Received message: {Channel} - {@Message}",
                channel,
                message);
        });
    }
}
```

最后，当我们同时运行 **Producer** 和 **Consumer** 服务时，得到的结果如下：

## 分布式系统中的缓存失效

在最近的一个项目中，我解决了一个分布式系统中的常见挑战：保持缓存同步。我们采用了两级缓存策略。首先，在每个 Web 服务器上有一个内存缓存，以实现超快速访问。其次，我们有一个共享的 Redis 缓存，以避免频繁访问数据库。

问题在于，当数据库中的数据发生变化时，我们需要一种方法快速通知所有 Web 服务器清除其内存缓存。这时，Redis 发布/订阅机制派上了用场。我们专门为缓存失效消息设置了一个 Redis 通道。

每个应用程序都会运行一个 **CacheInvalidationBackgroundService** ，订阅来自缓存失效通道的消息。

```java
public class CacheInvalidationBackgroundService(
    IServiceProvider serviceProvider)
    : BackgroundService
{
    public const string Channel = "cache-invalidation";

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await subscriber.SubscribeAsync(Channel, (channel, key) =>
        {
            var cache = serviceProvider.GetRequiredService<IMemoryCache>();

            cache.Remove(key);

            return Task.CompletedTask;
        });
    }
}
```

每当数据库中的数据发生变化时，我们会在该通道上发布一条消息，包含更新数据的缓存键。所有 Web 服务器都订阅了这个通道，因此它们可以立即知道从其内存缓存中移除旧数据。由于如果应用程序未运行，内存缓存会被清除，因此丢失缓存失效消息并不是问题。这保证了我们的缓存一致性，并确保用户始终看到最新的信息。

## 总结

Redis Pub/Sub 并非适用于所有消息传递需求的万能解决方案，但其简洁性和速度使其成为一个宝贵的工具。通道使我们能够轻松实现松散耦合组件之间的通信。

Redis 通道具有至多一次传递的语义，因此它们最适合偶尔丢失消息可以接受的情况。

我使用它来解决跨多个服务器同步缓存的挑战。这使得我们的系统能够提供最新的数据，而不会牺牲性能。

今天就到这里。期待与你再次相见。
