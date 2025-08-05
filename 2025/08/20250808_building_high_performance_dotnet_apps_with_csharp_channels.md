# [使用 C# 通道构建高性能 .NET 应用](https://antondevtips.com/blog/building-high-performance-dotnet-apps-with-csharp-channels)

构建可靠、可扩展且高性能的.NET 应用程序通常取决于你如何处理并发和数据处理。C# 通道为在 .NET 中构建安全、异步和高吞吐量的管道带来了一种新的现代化方法。

通道允许你创建内存中的生产者-消费者队列，这些队列可以自然地跨异步工作流和后台服务进行扩展。然而，一个关键的架构决策是在有界通道和无界通道之间做出选择。

在今天的文章中，我们将探讨：

- 什么是 C# Channels？
- 有界通道 vs. 无界通道
- 使用通道进行后台处理
- 在 ASP.NET Core 实际应用程序中使用通道
- 使用通道的最佳实践和技巧

让我们开始吧！

## 什么是 C# 通道？

在构建.NET 应用程序时，你经常需要将数据从代码的一部分发送到另一部分。

过去，开发者们使用 Queue\<T\>、ConcurrentQueue\<T\>或 BlockingCollection\<T\>等构造来实现这一目的。他们将这些队列封装到类中，并用它们来管理数据流。

然而，这种实现有一个显著的缺点：代码紧密耦合。

C# 通道解决了这个问题。它们实现了生产者-消费者模式。一个类生成数据，另一个类消费数据，而彼此互不了解。

C# 通道来自 System.Threading.Channels 命名空间。通道使生产者和消费者之间发送数据变得简单，帮助你避免常见的线程问题。

通道有两个部分：

- **写入**：将数据推送到通道中。
- **读取**：从通道中拉取数据。

读写操作可以在不同的线程上进行，而通道确保线程安全。它们让你可以在任何地方使用异步代码，这样你的应用程序就能处理大量数据而无需阻塞线程或使用锁。

以下是 C# 中使用通道的基本示例。在这里，我们异步生成数字并从通道中消费它们：

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

var channel = Channel.CreateUnbounded<int>();

// 生产者
_ = Task.Run(async () =>
{
    for (var i = 0; i < 10; i++)
    {
        await channel.Writer.WriteAsync(i);
        Console.WriteLine($"Produced: {i}");
        await Task.Delay(100); //  模拟处理
    }
    channel.Writer.Complete();
});

// 消费者
await foreach (var item in channel.Reader.ReadAllAsync())
{
    Console.WriteLine($"Consumed: {item}");
    await Task.Delay(150); // 模拟处理
}

Console.WriteLine("Processing complete.");
```

以下是它的工作原理：

- 生产者在单独的线程（任务）上运行，将数据写入通道。
- 消费者从通道读取数据并处理它。
-通道处理所有通信和线程安全。

通道在以下情况下非常适合使用：

- 你需要使用 async/await 来连接生产者和消费者。
- 你希望将生产和消费逻辑解耦
- 你想要控制数据流（例如，如果消费者跟不上速度，就减慢生产者的速度）。
- 你希望避免低级线程、锁或手动同步。

通道非常适合流式事件和处理后台任务。它们是单个应用程序中消息队列的简单内存替代方案。

## 有界通道 vs. 无界通道

C# 通道有两种类型：有界和无界。两者都允许你将数据从生产者传递给消费者，但它们处理流控制和内存的方式不同。

### 什么是有界通道？

有界通道具有固定的最大容量。创建时，你可以设置它能同时容纳的项数限制。

如果生产者在通道已满后尝试添加更多项目，它必须等待直到有可用空间。

什么时候应该使用有界通道？

- 当你想要限制内存使用并防止过载时。
- 当消费者有时比生产者慢时。
- 当你需要背压来避免系统过载时。

```csharp
var channel = Channel.CreateBounded<int>(5);

// 生产者
await channel.Writer.WriteAsync(1); // 如果有空间则添加项目，如果已满则等待

// 消费者
var item = await channel.Reader.ReadAsync(); // 获取一个项目
```

有界通道是后台处理、作业队列以及任何需要控制资源使用的应用程序的良好选择。它们有助于保护你的应用程序免受突然的峰值和失控的生产者的影响。

### 什么是无界通道？

无界通道没有固定限制。生产者可以按照他们想要的速度不断添加项目。

通道会增长以处理你放入其中的任意数量的项目，仅受可用系统内存的限制。

什么时候应该使用无界通道？

- 当你确定生产者永远不会长时间超过消费者的速度时。
- 对于流量控制不是问题的简单情况。
- 当你预期只有少量或稳定的项目流时。

以下是如何创建无界通道：

```csharp
var channel = Channel.CreateUnbounded<int>();

// 生产者
await channel.Writer.WriteAsync(42); // 始终接受新项目

// 消费者
var item = await channel.Reader.ReadAsync(); // 获取一个项目
```

无界通道使用简单，但在高负载情况下可能会有风险。如果生产者写入项目的速度比消费者读取它们的速度快，你可能会耗尽内存。

你应该使用哪一个？

如果你想要系统中的安全性和稳定性，请从有界通道开始。如果消费者处理速度跟不上，它们可以保护你的应用程序。

如果你知道生产者始终处于控制状态且数据速率较低，你可以使用无界通道。

在大多数实际应用中的.NET 服务中，有界通道是更安全的默认选择。

## 使用有界通道的后台处理器

在生产环境中，通道通常用于 ASP.NET Core 后台服务。

一个常见的模式是设置一个后台服务，从通道中读取消息或任务并逐个处理它们。这使得你的代码易于扩展，并保持主线程可用于其他工作。

让我们探索一个处理通道中项目的 **BackgroundService** 示例：

```csharp
builder.Services.AddSingleton(_ => Channel.CreateBounded<string>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
}));

public class MessageProcessor : BackgroundService
{
    private readonly Channel<string> _channel;
    private readonly ILogger<MessageProcessor> _logger;

    public MessageProcessor(Channel<string> channel, ILogger<MessageProcessor> logger)
    {
       _channel = channel;
       _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
       _logger.LogInformation("Message processor starting");

       await foreach (var message in _channel.Reader.ReadAllAsync(stoppingToken))
       {
          _logger.LogInformation("Processing message: {Message}", message);

          await Task.Delay(100, stoppingToken);

          _logger.LogInformation("Message processed successfully: {Message}", message);
       }
    }
}
```

要将项目发布到通道中，你需要调用 **WriteAsync** 方法：

```csharp
private Task AddSampleMessagesAsync(CancellationToken stoppingToken)
{
    _ = Task.Run(async () =>
    {
        // 添加150条消息（超过通道容量100）
        for (var i = 1; i <= 150 && !stoppingToken.IsCancellationRequested; i++)
        {
            var message = $"Sample message #{i}";

            // 等待通道中有空间（当通道已满时会阻塞）
            await _channel.Writer.WriteAsync(message, stoppingToken);

            _logger.LogInformation("Added message to channel: {Message}", message);

            await Task.Delay(50, stoppingToken);
        }

        // 完成通道的写入
        _channel.Writer.Complete();
    }, stoppingToken);
    return Task.CompletedTask;
}
```

请注意，我们在 **AsyncEnumerable** 上运行 foreach 循环：

```csharp
await foreach (var message in _channel.Reader.ReadAllAsync(stoppingToken))
{
    // 处理消息
}
```

**_channel.Reader.ReadAllAsync()** 等待通道中出现新消息。发布完消息后，你可以调用 **_channel.Writer.Complete()** ，循环将会结束。

通道的容量限制为 100，因此如果已有 100 条消息在等待，生产者将暂停，直到有空闲位置可用。后台服务会以其能够处理的最快速度读取消息。

如果你尝试过快地添加消息，生产者将会减速，这有助于控制你的内存使用量。

创建有界通道时，你可以设置 BoundedChannelFullMode 来控制通道满时发生的情况：

- **Wait**：写入方会等待直到有可用空间（最常见，最安全）。
- **DropWrite**：如果通道已满，新项目将被丢弃。
- **DropOldest**：移除最旧的项以腾出空间给新项。
- **DropNewest**：改为丢弃最新的项目。

对于大多数后台任务，请使用 **Wait**。当丢失少量消息是可以接受的情况下， **DropWrite** 或 **DropOldest** 可能是合适的选择。当最新事件比旧事件更重要时，你可以使用 **DropOldest** 。

## 使用通道的实际应用

在实际的 ASP.NET Core 应用程序中，你可以使用通道来实现写回缓存策略。

这种缓存策略用于高速、写入密集型场景。

主要思想是数据首先被写入缓存。缓存会在特定条件或时间间隔后，异步地将数据写回数据库。

让我们来探索一个真实世界的示例：一个在线商店应用程序。

用户经常在他们的在线购物车中添加或移除商品。最终状态仅在结账时才真正关键。写入操作可以非常快，因为它们首先命中缓存，系统可以定期将更新刷新到数据库。

这使得在购物高峰期间能够实现高写入吞吐量，而数据库最终会收到购物车的最终详细信息。

以下是创建产品购物车的 **WebApi** 端点：

```csharp
public record ProductCartRequest(string UserId, List<ProductCartItemRequest> ProductCartItems);

public record ProductCartItemRequest(Guid ProductId, int Quantity);

[HttpPost]
public async Task<ActionResult<ProductCartResponse>> CreateCart(ProductCartRequest request)
{
    var response = await _service.AddAsync(request);
    return CreatedAtAction(nameof(GetCart), new { id = response.Id }, response);
}
```

当创建一个新的 **ProductCart** 时，它仅被立即添加到缓存中：

```csharp
public class WriteBackCacheProductCartService
{
    private readonly HybridCache _cache;
    private readonly IProductCartRepository _repository;
    private readonly Channel<ProductCartDispatchEvent> _channel;

    public WriteBackCacheProductCartService(
        HybridCache cache,
        IProductCartRepository repository,
        Channel<ProductCartDispatchEvent> channel)
    {
        _cache = cache;
        _repository = repository;
        _channel = channel;
    }

    public async Task<ProductCartResponse> AddAsync(ProductCartRequest request)
    {
        var productCart = new ProductCart
        {
            Id = Guid.NewGuid(),
            UserId = request.UserId,
            CartItems = request.ProductCartItems.Select(x => new CartItem
            {
                Id = Guid.NewGuid(),
                Quantity = x.Quantity,
                Price = Random.Shared.Next(100, 1000)
            }).ToList()
        };

        var cacheKey = $"productCart:{productCart.Id}";

        var productCartResponse = MapToProductCartResponse(productCart);
        await _cache.SetAsync(cacheKey, productCartResponse);

        await _channel.Writer.WriteAsync(new ProductCartDispatchEvent(productCart));

        return productCartResponse;
    }
}    
```

这里我们使用一个有界通道来发布 **ProductCartDispatchEvent** 。

```csharp
public record ProductCartDispatchEvent(ProductCart ProductCart);

builder.Services.AddSingleton(_ => Channel.CreateBounded<ProductCartDispatchEvent>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
}));
```

一个后台任务从通道读取数据，并将购物车数据异步写入数据库：

```csharp
public class WriteBackCacheBackgroundService(IServiceScopeFactory scopeFactory,
    Channel<ProductCartDispatchEvent> channel) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var command in channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = scopeFactory.CreateScope();
            var repository = scope.ServiceProvider.GetRequiredService<IProductCartRepository>();

            var existingCart = await repository.GetByIdAsync(command.ProductCart.Id);
            if (existingCart is null)
            {
                await repository.AddAsync(command.ProductCart);
                return;
            }

            existingCart.CartItems = command.ProductCart.CartItems;
            await repository.UpdateAsync(existingCart);
        }
    }
}
```

> 注意：此模式可显著提高写入速度，但需要强大的冲突解决和故障处理机制来确保一致性。确保在缓存故障时购物车状态不会丢失，需要强大的复制或备份机制。

## 使用通道的最佳实践和技巧

通道是一个强大的工具，但要充分利用它们（并避免问题），遵循几个关键的最佳实践非常重要。以下是在.NET 应用程序中使用通道的一些实用技巧。

### 1. 为安全起见，优先使用有界通道

大多数情况下，你应该从有界通道开始。有界通道设置了可以等待处理的数据量限制。这使你的应用程序在重负载下更加稳定。

将容量设置为适合你工作负载的合理数值。如果你预期数据会出现峰值，请选择一个能够覆盖典型突发量但又不会过大的值。

无界通道只有在你知道数据速率始终较低时才是安全的。

### 2. 记得完成通道

当生产者完成写入时，在通道上调用 **channel.Writer.Complete()** 。这让消费者知道将不再有数据，使其能够完成处理。

如果你不完成通道，你的消费者的 ReadAllAsync 循环将永远不会结束。

### 3. 在写入和读取操作中使用 await

通道是为异步代码设计的。在写入或读取时始终使用 await 以避免阻塞线程并保持应用程序的响应性。

```csharp
await Channel.Writer.WriteAsync(data, cancellationToken);
await foreach (var item in channel.Reader.ReadAllAsync(cancellationToken))
{
    // 处理项目
}
```

### 4. 正确处理取消操作

在读取或写入通道时，始终传递一个 **CancellationToken** 。这允许你在应用程序关闭或用户取消操作时停止处理。

```csharp
await channel.Writer.WriteAsync(message, cancellationToken);
```

### 5. 不要在过多的生产者或消费者之间共享通道

虽然通道可以处理多个写入者和读取者，但过多的情况会使调试变得困难。在大多数情况下，坚持使用单个生产者和单个消费者可以获得最佳性能和最简单的推理。

如果你需要更多功能，.NET 通道支持多个读取器和写入器，但你应该仔细设计以避免意外情况。

### 6. 监控你的通道使用情况

在生产环境中注意你的通道容量。如果你经常看到通道被填满（生产者等待写入），这可能意味着你的消费者处理速度太慢。你可能需要加快处理速度或增大通道的大小。

### 7. 选择合适的 FullMode（对于有界通道）

创建有界通道时，你可以设置 **BoundedChannelFullMode** 来控制通道已满时发生的行为。

今天就到这里。期待与你再次相见。
