# [5 种在生产环境中真正有效的 .NET 并发模式](https://levelup.gitconnected.com/5-net-concurrency-patterns-that-actually-scale-in-production-37823cad3c4b)

大多数 .NET 应用程序不是因为 CPU 限制而失败；它们是因为糟糕的并发处理而失败。以下是 5 种在生产环境中真正可扩展的模式。

## 为什么大多数 .NET 性能问题不是硬件问题

我调试过足够多的生产环境故障，所以知道这一点：线程池耗尽导致的 .NET API 崩溃比 CPU 限制要多得多。

症状总是一样的。你的应用在轻负载下运行正常，然后在 500-1000 个并发请求时突然崩溃。CPU 使用率停留在 30%。内存看起来正常。但响应时间飙升至 30 秒以上，请求开始超时。

罪魁祸首是什么？通常是一个同步调用阻塞了异步线程。一个放在错误位置的 .Result 或 .Wait() 可能会级联成完全的线程池耗尽。我见过这种模式摧毁了本应轻松处理 10 倍负载的系统。

并发模式决定了你的扩展上限，而非服务器规格。选择错误的模式，你会在令人尴尬的低流量水平下遇到瓶颈。选择正确的模式，你的系统将能够平稳扩展，直到真正达到资源限制。

以下是五种在大规模应用中始终有效的并发模式，以及每种模式最重要的特定场景。

## 1. Async/Await：保持服务器正常运转的基础

为何重要：你应用中的每个 I/O 操作——数据库查询、HTTP 调用、文件读取——要么阻塞线程，要么释放线程以处理其他工作。这个选择决定了你的应用是能够线性扩展还是遇到瓶颈。

### Async/Await 的实际工作原理

![Async/await 在 I/O 操作期间释放线程，而阻塞调用则会耗尽线程池。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*LB4-XgfDv2Rrf4k1xA_ebw.png)

关键区别在于：Async 在 I/O 操作期间完全释放线程。当数据库响应时，任何线程都可以接续执行。

### 生产环境经验总结

在我们的金融科技 API 中，从同步 EF Core 切换到异步消除了线程池饥饿，并让我们在相同硬件上将每秒请求数从 800 扩展到 3000。数据库成为了我们的瓶颈，而不是应用程序线程。

常见陷阱：在异步上下文中使用 .Result 或 .Wait() 会导致死锁和线程饥饿。

### 扼杀性能的代码

```csharp
// 错误：阻塞线程，导致死锁
public IActionResult GetUserBalance(int userId)
{
    var user = _context.Users.FindAsync(userId).Result; // 线程杀手
    return Ok(user.Balance);
}

// 正确：释放线程以处理其他请求
public async Task<IActionResult> GetUserBalance(int userId)
{
    var user = await _context.Users.FindAsync(userId);
    return Ok(user.Balance);
}
```

常见陷阱：切勿使用 async void - 异常会丢失

```csharp
// 错误：异常会丢失
public async void ProcessPayment() => await _service.ProcessAsync();

// 正确：正确的错误传播
public async Task ProcessPayment() => await _service.ProcessAsync();
```

## 2. Channels：真正有效的背压机制

为何重要：像 ConcurrentQueue\<T\> 或 BlockingCollection\<T\> 这样的传统队列在负载情况下要么会丢弃工作，要么会消耗无限内存。Channels 为你提供具有可配置背压的有界队列——这正是后台作业所需要的。

### Channel 背压流程

![有界通道创建自然的背压，防止在流量激增时发生内存不足崩溃。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ObpPdpPcib6vYcqw5qPQyA.png)

神奇之处在于：有界通道无需复杂的限流逻辑就能自动创建回压。你的系统在负载下能够自我调节。

### 生产环境经验教训

在我设计的一个 SaaS 平台上，我们的 webhook 传递系统使用了无界队列。流量激增时会分配数百万条 webhook 消息，导致我们的 pod 因 OOM 错误而崩溃。切换到具有背压的有界通道后，内存使用保持平稳，并自然地对突发流量进行了速率限制。

最佳实践：在生产环境中始终使用有界通道，以防止内存爆炸。

### 有界通道设置

```csharp
// 在依赖注入中配置有界通道
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait // 注意：使用阻塞模式防止 OOM
});
services.AddSingleton(channel);
```

### 生产者模式

```csharp
// HTTP 处理程序 - 不要阻塞请求线程
public async Task<IActionResult> QueueWorkAsync(WorkItem item)
{
    // 触发并忘记模式以避免请求线程阻塞
    _ = Task.Run(async () => 
    {
        try
        {
            await _channelWriter.WriteAsync(item);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to queue work item {Id}", item.Id);
        }
    });
    
    return Accepted(); // 马上响应
}

// 对于后台服务 - 阻塞是可以接受的
public async Task<bool> QueueWorkDirectAsync(WorkItem item)
{
    await _channelWriter.WriteAsync(item); // 当通道已满时可以阻塞
    return true;
}
```

上下文很重要：在有界通道上调用 WriteAsync 时要小心。如果从 HTTP 请求线程调用，背压可能会导致请求延迟。使用即发即弃模式或将通道写入操作移出请求路径。

### 消费者模式

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await foreach (var item in _channelReader.ReadAllAsync(stoppingToken))
    {
        await ProcessWorkItemAsync(item, stoppingToken);
    }
}
```

### 为什么有界通道会产生自然背压

使用 FullMode.Wait 时，当队列填满时，生产者会自然减速而不是导致应用程序崩溃。这会产生自动背压，使系统在负载激增时保持稳定。

## 3. System.IO.Pipelines：无 GC 压力的流处理

为何重要：传统的流处理会重复复制缓冲区，导致大量垃圾回收开销。在高吞吐量场景下——如 WebSocket 数据流、TCP 服务器、消息代理——这会成为你的性能瓶颈。

### Pipeline 与传统流处理

![Pipelines 重用缓冲池，避免了因重复分配而导致的 GC 压力。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*PR9qjdkK1TegDCx_7ZDsLA.png)

性能差异：传统流为每次读取操作创建新的字节数组。Pipelines 使用池化的可重用内存段。

### 生产环境经验教训

我在一个每分钟处理 50,000 笔交易的支付网关上吃尽了苦头才吸取了这个教训。我们最初采用的 naive Stream.ReadAsync 方法每小时分配 2GB 的临时缓冲区。切换到 Pipelines 后，分配量减少了 65%，并在高峰时段消除了 GC 暂停。

最佳实践：在任何高吞吐量流式处理场景中使用 Pipelines，以消除分配开销。

### 零分配流解析

```csharp
public async Task ReadMessagesAsync(PipeReader reader, CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        var result = await reader.ReadAsync(ct);
        var buffer = result.Buffer;

        // 处理完整消息而不复制缓冲区
        while (TryParseMessage(ref buffer, out var message))
        {
            await HandleMessageAsync(message, ct);
        }

        reader.AdvanceTo(buffer.Start, buffer.End);
        if (result.IsCompleted) break;
    }
}
```

### 高效消息解析器

```csharp
private bool TryParseMessage(ref ReadOnlySequence<byte> buffer, out Message message)
{
    if (buffer.Length < 4) { message = default; return false; }
    
    // 零分配：使用 BinaryPrimitives 而不是 ToArray()
    var lengthSpan = buffer.Slice(0, 4);
    int length;
    
    if (!lengthSpan.IsSingleSegment)
    {
        Span<byte> temp = stackalloc byte[4];
        lengthSpan.CopyTo(temp);
        length = BinaryPrimitives.ReadInt32LittleEndian(temp);
    }
    else
    {
        length = BinaryPrimitives.ReadInt32LittleEndian(lengthSpan.FirstSpan);
    }
    
    if (buffer.Length < 4 + length) { message = default; return false; }
    
    message = new Message(buffer.Slice(4, length).ToArray()); // 按需分配
    buffer = buffer.Slice(4 + length);
    return true;
}
```

常见陷阱：在小范围上使用 .ToArray() 会抵消 Pipeline 的零分配优势。解析整数和基元类型时使用 BinaryPrimitives 。

### 为什么 Pipeline 在大规模应用中胜出

Pipeline 使用池化内存并避免中间分配。同样的代码，如果使用 Stream.ReadAsync 每秒会创建数千个临时字节数组，而正确使用 Pipeline 则可以实现零分配。

## 4. Actor 模式：通过隔离状态实现扩展

为何重要：基于锁的并发是错误滋生之源和性能杀手。Actor 模式通过为每个实体提供其自己的消息队列和状态来解决这个问题。由于一次只处理一条消息，因此你完全不需要锁。

### Actor 消息处理流程

![Actor 一次处理一条消息，从而消除了围绕共享状态的锁争用。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*iJiyxCOBwAvhMU5XmVHGvQ.png)

关键洞察：多个客户端可以并发访问 Actor，但 Actor 会按顺序处理消息，从而无需使用锁。

### 生产经验教训

在一个多租户 SaaS 平台上，我们的共享字典查找由于锁争用导致请求堆积。迁移到每个租户一个 Orleans 执行元模型后，消除了所有锁开销，并在峰值负载下将吞吐量提高了一倍。

最佳实践：对有状态实体（用户会话、账户、游戏对象）使用执行元模式，以消除锁争用。

### 简单执行元模式（教学示例）

```csharp
public class AccountActor
{
    private readonly Channel<IAccountMessage> _messages = Channel.CreateUnbounded<IAccountMessage>();
    private decimal _balance;
    
    public AccountActor() => _ = ProcessMessagesAsync();

    public async Task<decimal> GetBalanceAsync()
    {
        var tcs = new TaskCompletionSource<decimal>();
        await _messages.Writer.WriteAsync(new GetBalanceMessage(tcs));
        return await tcs.Task;
    }

    private async Task ProcessMessagesAsync()
    {
        await foreach (var msg in _messages.Reader.ReadAllAsync())
        {
            // 每次只处理一条消息 - 无需锁
            if (msg is GetBalanceMessage getBalance)
                getBalance.Result.SetResult(_balance);
        }
    }
}
```

生产环境注意事项：以上内容仅供教学参考。在生产系统中，请使用经过验证的框架，如 Orleans（微软的虚拟 actor 框架）或 Akka.NET，而不是自行构建 actor 系统。

### 可用于生产环境的 Actor 框架

对于企业级系统，可以考虑以下经过验证的框架：

- Orleans：微软的虚拟 Actor 框架，应用于 Xbox Live 和 Azure 服务
- Proto.Actor：轻量级、受 Go 启发的 Actor 模型
- Akka.NET：功能全面，在 JVM 生态系统中久经考验

Actor 的局限性：Actor 并非万能良药。它们存在消息分发的开销，如果单个 Actor 成为热点，可能会造成瓶颈。考虑采用分区策略（如用户 ID 分片、地理分布）来避免单个 Actor 的瓶颈问题。此外，要警惕复杂的 Actor 依赖链，它们可能引发延迟级联。

常见陷阱：不要为简单的只读场景构建 Actor —— 如果没有可变状态，消息开销就不值得。

## 5. Parallel.ForEachAsync：利用所有核心的 CPU 工作

为何重要：对于 CPU 密集型任务，你希望使用所有可用的核心。Parallel.ForEachAsync（在 .NET 6 中引入）可以安全地并行处理 CPU 密集型工作和异步 I/O 操作。

### Parallel.ForEachAsync 执行流程

![Parallel.ForEachAsync 将工作分散到所有核心上，结合了异步 I/O 和 CPU 密集型处理。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*BoashiONqOYs0YXNmTR4hA.png)

执行模型：每个并行任务处理自己的异步 I/O 操作，而 CPU 工作则分布在可用核心上。任务在完成时自动获取新的工作项。

### 生产经验教训

我在一个需要每天调整 50 万张图像大小的批处理作业中使用了这种模式。顺序处理需要 7 小时。在一台 8 核机器上，并行处理将其缩短至 90 分钟，且无需编写复杂的线程代码。

最佳实践：对结合 I/O 操作和 CPU 处理的混合工作负载使用 Parallel.ForEachAsync 。

### CPU + I/O 并行处理

```csharp
// 利用所有 CPU 核心处理图像
await Parallel.ForEachAsync(imagePaths, 
    new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
    async (imagePath, ct) =>
    {
        var imageData = await File.ReadAllBytesAsync(imagePath, ct); // I/O 操作
        var processed = ResizeImage(imageData); // CPU 操作 
        await File.WriteAllBytesAsync($"{imagePath}.out", processed, ct); // I/O 操作
    });
```

### 混合 I/O + CPU 工作负载

```csharp
// 下载并处理 URL
await Parallel.ForEachAsync(urls, 
    new ParallelOptions { MaxDegreeOfParallelism = 10 }, // 限制并发下载
    async (url, ct) =>
    {
        try
        {
            var content = await _httpClient.GetByteArrayAsync(url, ct); // I/O 操作
            var processed = ProcessContent(content); // CPU 操作
            await SaveResultAsync(processed, ct); // I/O 操作
        }
        catch (Exception ex)
        {
            // 处理单个失败 - 异常不会传播到其他任务
            _logger.LogError(ex, "Failed to process {Url}", url);
            // 考虑收集错误以进行批量报告
        }
    });
```

I/O 密集型工作负载警告：对于纯 I/O 任务（如大量 HTTP 调用、数据库查询）， Parallel.ForEachAsync 可能效率低下，因为每次迭代在等待期间都会占用一个线程池线程。请考虑使用 SemaphoreSlim + Task.WhenAll 替代：

```csharp
// 更高效的 I/O 密集型工作负载
var semaphore = new SemaphoreSlim(10); // 限制并发数
var tasks = urls.Select(async url =>
{
    await semaphore.WaitAsync();
    try
    {
        return await _httpClient.GetByteArrayAsync(url);
    }
    finally 
    {
        semaphore.Release();
    }
});

var results = await Task.WhenAll(tasks);
```

异常行为： Parallel.ForEachAsync 在遇到第一个未处理异常时会停止，并且只会抛出该异常。如果需要收集所有失败信息，请在每次迭代内部处理异常或使用聚合模式。

### 针对不同工作负载的调优

最佳实践：根据工作负载特性调整 MaxDegreeOfParallelism ：

- CPU 密集型： Environment.ProcessorCount
- I/O 密集型：根据外部服务限制使用更高的值（10–50）
- 混合工作负载：从 ProcessorCount * 2 开始并进行测量

## 选择合适的模式：快速决策指南

Web API 和请求处理：始终在所有地方使用 async/await。

后台作业队列：使用具有有界容量和 FullMode.Wait 的 Channels。

高吞吐量流处理：将 System.IO.Pipelines 用于 WebSockets、TCP 服务器或消息处理。

有状态多用户系统：对于游戏、实时协作或多租户 SaaS，考虑使用 Actor 模式（生产环境可使用 Orleans/Akka.NET）。

CPU 密集型批处理作业：将 Parallel.ForEachAsync 用于：

- MaxDegreeOfParallelism = Environment.ProcessorCount 用于 CPU 密集型工作
- I/O 密集型工作负载的较高值（10-50）
- ProcessorCount * 2 适用于混合工作负载（需测量并调整）

### 生产架构示例

我们的金融科技平台结合了这些模式：

- 针对 Web API 使用 async/await
- 用于具有有限容量（1000 个项目， FullMode.Wait ）的支付处理队列的 Channels
- 用于实时交易数据流的 Pipelines
- 用于账户状态管理的 Orleans Actor 模型
- 用于夜间批量对账的 Parallel.ForEachAsync

每种模式都解决了特定的扩展瓶颈。它们共同构建了既能水平扩展又能垂直扩展的系统。

## 立即采取行动

首先，检查你当前的代码库是否存在这些反模式：

- Web 控制器中的同步数据库调用（阻塞线程）
- 无界 Channel 可能消耗无限内存
- 流处理创建过多的临时分配
- 围绕共享状态的锁争用
- 可并行化 CPU 工作的顺序处理

一次修复一个模式，衡量其影响，并建立对这些方法的信心。

在下一个系统设计中，收藏本指南并根据你的扩展需求预先选择模式。

今天就到这里。期待与你再次相见。
