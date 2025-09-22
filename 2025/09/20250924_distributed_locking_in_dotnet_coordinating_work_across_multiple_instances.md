# [.NET 中的分布式锁：跨多实例协调工作](https://www.milanjovanovic.tech/blog/distributed-locking-in-dotnet-coordinating-work-across-multiple-instances)

当你构建跨多个服务器或进程运行的应用程序时，你最终会遇到并发访问的问题。多个工作线程尝试同时更新同一资源，最终导致竞争条件、重复工作或数据损坏。

.NET 为单进程场景提供了出色的并发控制原语，如 lock 、 SemaphoreSlim 和 Mutex 。但是当你的应用程序扩展到多个实例时，这些原语就不再起作用了。

分布式锁应运而生。

分布式锁通过确保一次只有一个节点（应用程序实例）可以访问关键部分来提供解决方案，从而防止竞态条件并在整个分布式系统中维护数据一致性。

## 为什么以及何时需要分布式锁

在单进程应用中，你可以使用 lock 或 .NET 10 中的新 Lock 类。但一旦你进行扩展，这就不够了，因为每个进程都有自己的内存空间。

分布式锁有价值的一些常见情况：

- 后台作业：确保一次只有一个工作进程处理特定的作业或资源。
- 领导者选举：选择单个进程来执行定期工作（如应用异步数据库投影）。
- 避免重复执行：确保在部署到多个实例时，计划任务不会多次运行。
- 协调共享资源：例如，一次只有一个服务实例执行迁移或清理操作。
- 防止缓存雪崩：确保当给定的缓存键过期时，只有一个实例刷新缓存。

关键价值：在分布式环境中保持一致性和安全性。没有这一点，你将面临重复操作、状态损坏或不必要负载的风险。

现在你知道为什么分布式锁很重要了。

让我们来看一些实现选项。

## 使用 PostgreSQL 建议锁实现 DIY 分布式锁

让我们从简单的开始。PostgreSQL 有一个称为建议锁的功能，非常适合分布式锁。与表锁不同，这些锁不会干扰你的数据——它们纯粹用于协调。

这是一个示例：

```csharp
public class NightlyReportService(NpgsqlDataSource dataSource)
{
    public async Task ProcessNightlyReport()
    {
        await using var connection = dataSource.OpenConnection();

        var key = HashKey("nightly-report");

        var acquired = await connection.ExecuteScalarAsync<bool>(
            "SELECT pg_try_advisory_lock(@key)",
            new { key });

        if (!acquired)
        {
            throw new ConflictException("Another instance is already processing the nightly report");
        }

        try
        {
            await DoWork();
        }
        finally
        {
            await connection.ExecuteAsync(
                "SELECT pg_advisory_unlock(@key)",
                new { key });
        }
    }

    private static long HashKey(string key) =>
        BitConverter.ToInt64(SHA256.HashData(Encoding.UTF8.GetBytes(key)), 0);

    private static Task DoWork() => Task.Delay(5000); // 实际执行的业务
}
```

这是在幕后发生的情况。

首先，我们将锁名称转换为一个数字。PostgreSQL 建议锁需要数字键，因此我们将 nightly-report 哈希为一个 64 位整数。每个节点（应用程序实例）必须为相同的字符串生成相同的数字，否则这将无法工作。

接下来， pg_try_advisory_lock() 尝试获取该数字的排他锁。如果成功，它返回 true ；如果另一个连接已经持有该锁，则返回 false 。此调用不会阻塞 - 它会立即告诉你是否获得了锁。

如果我们获取到锁，就执行我们的工作。如果没有，我们就返回一个冲突响应，让其他实例来处理它。

finally 代码块确保我们总是会释放锁，即使出现问题。PostgreSQL 还会在连接关闭时自动释放咨询锁，这是一个很好的安全措施。

SQL Server 通过 sp_getapplock 提供了类似的功能。

## 探索 DistributedLock 库

虽然自己动手的方法可行，但生产应用程序需要更复杂的功能。DistributedLock 库处理了各种边缘情况，并提供了多种后端选项（Postgres、Redis、SqlServer 等）。你知道我是不重复造轮子的支持者，所以这是一个很好的选择。

安装该包：

```bash
Install-Package DistributedLock
```

我将使用 IDistributedLockProvider 的方法，它与依赖注入(DI)配合得很好。你可以在无需了解底层基础设施的情况下获取锁。你只需在 DI 容器中注册一个锁提供程序实现即可。

例如，使用 Postgres：

```csharp
// 注册Postgres锁提供程序
builder.Services.AddSingleton<IDistributedLockProvider>(
    (_) =>
    {
        return new PostgresDistributedSynchronizationProvider(
            builder.Configuration.GetConnectionString("distributed-locking")!);
    });
```

或者如果你想使用 Redis 并采用 Redlock 算法：

```csharp
// 需要注册 StackExchange.Redis
builder.Services.AddSingleton<IConnectionMultiplexer>(
    (_) =>
    {
        return ConnectionMultiplexer.Connect(
            builder.Configuration.GetConnectionString("redis")!);
    });

// 注册Redis锁提供程序
builder.Services.AddSingleton<IDistributedLockProvider>(
    (sp) =>
    {
        var connectionMultiplexer = sp.GetRequiredService<IConnectionMultiplexer>();

        return new RedisDistributedSynchronizationProvider(connectionMultiplexer.GetDatabase());
    });
```

用法非常简单：

```csharp
// 你也可以传入一个超时，该提供程序将重复尝试获取锁，直到超时。
IDistributedSynchronizationHandle? distributedLock = distributedLockProvider
    .TryAcquireLock("nightly-report");

// 假如没有获取锁，对象将变为空
if (distributedLock is null)
{
    return Results.Conflict();
}

// 注意：确保在完成工作后释放锁
using (distributedLock)
{
    await DoWork();
}
```

该库处理所有棘手的部分：超时、重试，以及确保即使在故障情况下也能释放锁。

它还支持多种后端（SQL Server、Azure、ZooKeeper 等），使其成为生产工作负载的可靠选择。

## 总结

分布式锁不是你每天都需要的东西。但当你需要它时，它能帮你避免那些只有在负载情况下或生产环境中才会出现的微妙且令人痛苦的问题。

从简单开始：如果你已经在使用 Postgres，那么建议锁是一个强大的工具。

为了获得更简洁的开发体验，请使用 DistributedLock 库。

选择适合你基础设施的后端（Postgres、Redis、SQL Server 等）。

在适当的时间使用正确的锁，可以确保你的系统保持一致性、可靠性和弹性，即使在多个进程和服务器之间也是如此。

今天就到这里。期待与你再次相见。
