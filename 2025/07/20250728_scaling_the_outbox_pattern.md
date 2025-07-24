# [扩展发件箱模式（每日20亿消息）](https://www.milanjovanovic.tech/blog/scaling-the-outbox-pattern)

昨天，我谈到了实现发件箱模式。这是可靠分布式消息传递的关键工具。但实现它只是第一步。

今天，我们将提升一个档次。我们将从一个基本的发件箱处理器开始，并将其转变为一个高性能引擎，能够每天处理超过20亿条消息。

闲话少说，立即开干！

## 起点

这是我们起点。我们有一个 **OutboxProcessor** ，它会轮询未处理的消息并将它们发布到队列中。我们首先可以做的就是调整轮询频率和批量大小。

```java
internal sealed class OutboxProcessor(NpgsqlDataSource dataSource, IPublishEndpoint publishEndpoint)
{
    private const int BatchSize = 1000;

    public async Task<int> Execute(CancellationToken cancellationToken = default)
    {
        await using var connection = await dataSource.OpenConnectionAsync(cancellationToken);
        await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

        var messages = await connection.QueryAsync<OutboxMessage>(
            @"""
            SELECT *
            FROM outbox_messages
            WHERE processed_on_utc IS NULL
            ORDER BY occurred_on_utc LIMIT @BatchSize
            """,
            new { BatchSize },
            transaction: transaction);

        foreach (var message in messages)
        {
            try
            {
                var messageType = Messaging.Contracts.AssemblyReference.Assembly.GetType(message.Type);
                var deserializedMessage = JsonSerializer.Deserialize(message.Content, messageType);

                await publishEndpoint.Publish(deserializedMessage, messageType, cancellationToken);

                await connection.ExecuteAsync(
                    @"""
                    UPDATE outbox_messages
                    SET processed_on_utc = @ProcessedOnUtc
                    WHERE id = @Id
                    """,
                    new { ProcessedOnUtc = DateTime.UtcNow, message.Id },
                    transaction: transaction);
            }
            catch (Exception ex)
            {
                await connection.ExecuteAsync(
                    @"""
                    UPDATE outbox_messages
                    SET processed_on_utc = @ProcessedOnUtc, error = @Error
                    WHERE id = @Id
                    """,
                    new { ProcessedOnUtc = DateTime.UtcNow, Error = ex.ToString(), message.Id },
                    transaction: transaction);
            }
        }

        await transaction.CommitAsync(cancellationToken);

        return messages.Count;
    }
}
```

假设我们持续运行 **OutboxProcessor** 。我将批量大小增加到 **1000** 。

我们能处理多少条消息？

我将运行发件箱处理 **1 分钟**，并统计处理了多少条消息。

基线实现每分钟处理 **81,000** 条消息，即每秒 **1,350** 条消息（MPS）。

不错，但让我们看看还能改进多少。

## 衡量每一步

你不能改进你无法衡量的东西。对吧？所以，我会用 **Stopwatch** 来衡量总执行时间和每个步骤所需的时间。

请注意，我还将发布和更新步骤分开了。这样我可以分别测量发布和更新所需的时间。这在以后会很重要，因为我想分别优化每个步骤。

使用基线实现，以下是每个步骤的执行时间：

- 查询时间：约 70 毫秒
- 发布时间：约 320 毫秒
- 更新时间：约 300 毫秒

```java
internal sealed class OutboxProcessor(
    NpgsqlDataSource dataSource,
    IPublishEndpoint publishEndpoint,
    ILogger<OutboxProcessor> logger)
{
    private const int BatchSize = 1000;

    public async Task<int> Execute(CancellationToken cancellationToken = default)
    {
        var totalStopwatch = Stopwatch.StartNew();
        var stepStopwatch = new Stopwatch();

        await using var connection = await dataSource.OpenConnectionAsync(cancellationToken);
        await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

        stepStopwatch.Restart();
        var messages = (await connection.QueryAsync<OutboxMessage>(
            @"""
            SELECT *
            FROM outbox_messages
            WHERE processed_on_utc IS NULL
            ORDER BY occurred_on_utc LIMIT @BatchSize
            """,
            new { BatchSize },
            transaction: transaction)).AsList();
        var queryTime = stepStopwatch.ElapsedMilliseconds;

        var updateQueue = new ConcurrentQueue<OutboxUpdate>();

        stepStopwatch.Restart();
        foreach (var message in messages)
        {
            try
            {
                var messageType = Messaging.Contracts.AssemblyReference.Assembly.GetType(message.Type);
                var deserializedMessage = JsonSerializer.Deserialize(message.Content, messageType);

                await publishEndpoint.Publish(deserializedMessage, messageType, cancellationToken);

                updateQueue.Enqueue(new OutboxUpdate
                {
                    Id = message.Id,
                    ProcessedOnUtc = DateTime.UtcNow
                });
            }
            catch (Exception ex)
            {
                updateQueue.Enqueue(new OutboxUpdate
                {
                    Id = message.Id,
                    ProcessedOnUtc = DateTime.UtcNow,
                    Error = ex.ToString()
                });
            }
        }
        var publishTime = stepStopwatch.ElapsedMilliseconds;

        stepStopwatch.Restart();
        foreach (var outboxUpdate in updateQueue)
        {
            await connection.ExecuteAsync(
                @"""
                UPDATE outbox_messages
                SET processed_on_utc = @ProcessedOnUtc, error = @Error
                WHERE id = @Id
                """,
                outboxUpdate,
                transaction: transaction);
        }
        var updateTime = stepStopwatch.ElapsedMilliseconds;

        await transaction.CommitAsync(cancellationToken);

        totalStopwatch.Stop();
        var totalTime = totalStopwatch.ElapsedMilliseconds;

        OutboxLoggers.Processing(logger, totalTime, queryTime, publishTime, updateTime, messages.Count);

        return messages.Count;
    }

    private struct OutboxUpdate
    {
        public Guid Id { get; init; }
        public DateTime ProcessedOnUtc { get; init; }
        public string? Error { get; init; }
    }
}
```

做完这一步，接下来就要进入有趣的部分！

## 优化读取查询

首先我想优化的是获取未处理消息的查询。如果我们不需要所有列，执行一个 SELECT * 查询将会产生影响（提示：我们不需要）。

以下是当前的 SQL 查询：

```sql
SELECT *
FROM outbox_messages
WHERE processed_on_utc IS NULL
ORDER BY occurred_on_utc LIMIT @BatchSize
```

我们可以修改查询，只返回我们需要的列。这将为我们节省一些带宽，但不会显著提高性能。

```sql
SELECT id AS Id, type AS Type, content as Content
FROM outbox_messages
WHERE processed_on_utc IS NULL
ORDER BY occurred_on_utc LIMIT @BatchSize
```

让我们检查这个查询的执行计划。你会看到它正在执行表扫描。我正在 PostgreSQL 上运行这个，这是我从 EXPLAIN ANALYZE 得到的结果：

```bash
Limit  (cost=86169.40..86286.08 rows=1000 width=129) (actual time=122.744..124.234 rows=1000 loops=1)
  ->  Gather Merge  (cost=86169.40..245080.50 rows=1362000 width=129) (actual time=122.743..124.198 rows=1000 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=85169.38..86871.88 rows=681000 width=129) (actual time=121.478..121.492 rows=607 loops=3)
              Sort Key: occurred_on_utc
              Sort Method: top-N heapsort  Memory: 306kB
              Worker 0:  Sort Method: top-N heapsort  Memory: 306kB
              Worker 1:  Sort Method: top-N heapsort  Memory: 306kB
              ->  Parallel Seq Scan on outbox_messages  (cost=0.00..47830.88 rows=681000 width=129) (actual time=0.016..67.481 rows=666667 loops=3)
                    Filter: (processed_on_utc IS NULL)
Planning Time: 0.051 ms
Execution Time: 124.298 ms
```

现在，我将创建一个“覆盖”查询未处理消息的索引。覆盖索引包含满足查询所需的所有列，而无需访问表本身。

索引将建立在 *occurred_on_utc* 和 *processed_on_utc* 列上。它将包括 *id* 、 *type* 和 *content* 列。最后，我们将应用一个过滤器，仅对未处理的消息进行索引。

```sql
CREATE INDEX IF NOT EXISTS idx_outbox_messages_unprocessed
ON public.outbox_messages (occurred_on_utc, processed_on_utc)
INCLUDE (id, type, content)
WHERE processed_on_utc IS NULL
```

让我解释每个决策背后的原因：

- 对 *occurred_on_utc* 进行索引会将条目按升序存储在索引中。这与查询中的 *ORDER BY occurred_on_utc* 语句相匹配。这意味着查询可以扫描索引，而无需对结果进行排序。结果已经处于正确的排序顺序。
- 将我们选择的列包含在索引中，可以让我们直接从索引条目中返回这些列。这样可以避免从表行中读取这些值。
- 在索引中过滤未处理的消息满足 *WHERE processed_on_utc IS NULL* 声明。

注意：PostgreSQL 的最大索引行大小为 2712 字节（别问我怎么知道的）。 INCLUDE 列表中的列也是索引行（B 树元组）的一部分。 content 列包含序列化的 JSON 消息，因此它最有可能导致我们超过这个限制。没有办法绕过这一点，所以我的建议是尽量保持消息的大小尽可能小。你可以将这个列从 INCLUDE 列表中排除，以牺牲一点性能。

以下是创建此索引后的更新执行计划：

```bash
Limit  (cost=0.43..102.82 rows=1000 width=129) (actual time=0.016..0.160 rows=1000 loops=1)
  ->  Index Only Scan using idx_outbox_messages_unprocessed on outbox_messages  (cost=0.43..204777.36 rows=2000000 width=129) (actual time=0.015..0.125 rows=1000 loops=1)
        Heap Fetches: 0
Planning Time: 0.059 ms
Execution Time: 0.189 ms
```

由于我们有一个覆盖索引，执行计划只包含 Index Only Scan 和 Limit 操作。不需要进行过滤或排序，这就是我们看到性能大幅提升的原因。

查询时间会受到怎样的性能影响？

- 查询时间：70 毫秒 → 1 毫秒（减少 98.5%）

## 优化消息发布

我们可以优化的下一件事是如何将消息发布到队列。我使用 MassTransit 的 **IPublishEndpoint** 来发布到 RabbitMQ。

更准确地说，我们是在发布到交换机。交换机随后会将消息路由到相应的队列。

但是我们应该如何优化这一点呢？

我们可以进行的一个小小的优化就是引入一个用于序列化中使用的消息类型的缓存。不断为每个消息类型执行反射是昂贵的，所以我们将反射执行一次，并存储结果。

这里缓存可以是一个 **ConcurrentDictionary** ，我们将使用 GetOrAdd 来检索缓存的类型。

我将这段代码提取到 **GetOrAddMessageType** 辅助方法中：

```java
private static readonly ConcurrentDictionary<string, Type> TypeCache = new();

private static Type GetOrAddMessageType(string typeName)
{
    return TypeCache.GetOrAdd(
        typeName,
        name => Messaging.Contracts.AssemblyReference.Assembly.GetType(name));
}
```

这是我们消息发布步骤的样子。最大的问题是我们通过等待 **Publish** 来完成它。 **Publish** 需要一些时间，因为它在等待来自消息代理的确认。我们正在循环中做这件事，这使得效率更低。

```java
var updateQueue = new ConcurrentQueue<OutboxUpdate>();

foreach (var message in messages)
{
    try
    {
        var messageType = Messaging.Contracts.AssemblyReference.Assembly.GetType(message.Type);
        var deserializedMessage = JsonSerializer.Deserialize(message.Content, messageType);

        // 我们需要等待消息代理确认。
        await publishEndpoint.Publish(deserializedMessage, messageType, cancellationToken);

        updateQueue.Enqueue(new OutboxUpdate
        {
            Id = message.Id,
            ProcessedOnUtc = DateTime.UtcNow
        });
    }
    catch (Exception ex)
    {
        updateQueue.Enqueue(new OutboxUpdate
        {
            Id = message.Id,
            ProcessedOnUtc = DateTime.UtcNow,
            Error = ex.ToString()
        });
    }
}
```

我们可以通过批量发布消息来改进这一点。实际上， **IPublishEndpoint** 有一个 **PublishBatch** 扩展方法。如果我们深入查看，会发现以下内容：

```java
public static Task PublishBatch(
    this IPublishEndpoint endpoint,
    IEnumerable<object> messages,
    CancellationToken cancellationToken = default)
{
    return Task.WhenAll(messages.Select(x => endpoint.Publish(x, cancellationToken)));
}
```

所以我们可以将消息集合转换为一系列发布任务，然后使用 **Task.WhenAll** 进行等待。

```java
var updateQueue = new ConcurrentQueue<OutboxUpdate>();

var publishTasks = messages
    .Select(message => PublishMessage(message, updateQueue, publishEndpoint, cancellationToken))
    .ToList();

await Task.WhenAll(publishTasks);

// 我把消息发布到一个单独的方法中，以提高可读性。
private static async Task PublishMessage(
    OutboxMessage message,
    ConcurrentQueue<OutboxUpdate> updateQueue,
    IPublishEndpoint publishEndpoint,
    CancellationToken cancellationToken)
{
    try
    {
        var messageType = GetOrAddMessageType(message.Type);
        var deserializedMessage = JsonSerializer.Deserialize(message.Content, messageType);

        await publishEndpoint.Publish(deserializedMessage, messageType, cancellationToken);

        updateQueue.Enqueue(new OutboxUpdate
        {
            Id = message.Id,
            ProcessedOnUtc = DateTime.UtcNow
        });
    }
    catch (Exception ex)
    {
        updateQueue.Enqueue(new OutboxUpdate
        {
            Id = message.Id,
            ProcessedOnUtc = DateTime.UtcNow,
            Error = ex.ToString()
        });
    }
}
```

消息发布步骤的改进是什么？

- 发布时间：320 毫秒 → 289 毫秒（-9.8%）

如您所见，这并没有显著加快速度。但这是为了让我们能够从我所准备的其他优化中受益。

## 优化更新查询

我们优化之旅的下一步是解决更新已处理发件消息的查询问题。

当前实现效率低下，因为我们对每个发件消息都向数据库发送一个查询。

```java
foreach (var outboxUpdate in updateQueue)
{
    await connection.ExecuteAsync(
        @"""
        UPDATE outbox_messages
        SET processed_on_utc = @ProcessedOnUtc, error = @Error
        WHERE id = @Id
        """,
        outboxUpdate,
        transaction: transaction);
}
```

如果你现在还没收到消息，批处理就是关键。我们需要一种方法来向数据库发送一个大的 **UPDATE** 查询。

我们必须手动构建这个批量查询的 SQL。我们将使用 **Dapper** 中的 **DynamicParameters** 类型来提供所有参数。

```java
var updateSql =
    @"""
    UPDATE outbox_messages
    SET processed_on_utc = v.processed_on_utc,
        error = v.error
    FROM (VALUES
        {0}
    ) AS v(id, processed_on_utc, error)
    WHERE outbox_messages.id = v.id::uuid
    """;

var updates = updateQueue.ToList();
var paramNames = string.Join(",", updates.Select((_, i) => $"(@Id{i}, @ProcessedOn{i}, @Error{i})"));

var formattedSql = string.Format(updateSql, paramNames);

var parameters = new DynamicParameters();

for (int i = 0; i < updates.Count; i++)
{
    parameters.Add($"Id{i}", updates[i].Id.ToString());
    parameters.Add($"ProcessedOn{i}", updates[i].ProcessedOnUtc);
    parameters.Add($"Error{i}", updates[i].Error);
}

await connection.ExecuteAsync(formattedSql, parameters, transaction: transaction);
```

这将生成一个类似这样的 SQL 查询：

```sql
UPDATE outbox_messages
SET processed_on_utc = v.processed_on_utc,
    error = v.error
FROM (VALUES
    (@Id0, @ProcessedOn0, @Error0),
    (@Id1, @ProcessedOn1, @Error1),
    (@Id2, @ProcessedOn2, @Error2),
    -- A few hundred rows in beteween
    (@Id999, @ProcessedOn999, @Error999)
) AS v(id, processed_on_utc, error)
WHERE outbox_messages.id = v.id::uuid
```

而不是每条消息发送一个更新查询，我们可以发送一个查询来更新所有消息。

这显然会给我们带来显著的性能提升：

- 更新时间：300 毫秒 → 52 毫秒（-82.6%）

## 我们进展到什么程度了？

让我们测试一下当前优化后的性能提升。到目前为止，我们所做的更改主要集中在提高 **OutboxProcessor** 的速度上。

以下是我在各个步骤中看到的大致数字：

- 查询时间：~1 毫秒
- 发布时间：约 289 毫秒
- 更新时间：约 52 毫秒

我将运行发件箱处理 **1分钟**，并计算已处理消息的数量。

优化后的实现每分钟处理 **162,000** 条消息，即每秒 **2,700** 条消息。

这使我们能够每天处理超过 **2.3亿** 条消息。

但我们才刚刚开始。

## 并行发件处理

如果我们想进一步发展，我们必须扩展 **OutboxProcessor** 。我们可能面临的问题是多次处理同一条消息。因此，我们需要对当前批次的消息实现某种形式的锁定。

PostgreSQL 有一个方便的 **FOR UPDATE** 语句，我们可以在这里使用。它会在当前事务的持续期间锁定所选行。然而，我们必须添加 **SKIP LOCKED** 语句，以允许其他查询跳过被锁定的行。否则，任何其他查询都会被阻塞，直到当前事务完成。

以下是更新后的查询：

```sql
SELECT id AS Id, type AS Type, content as Content
FROM outbox_messages
WHERE processed_on_utc IS NULL
ORDER BY occurred_on_utc LIMIT @BatchSize
FOR UPDATE SKIP LOCKED
```

要扩展 **OutboxProcessor** ，我们只需运行多个后台作业实例。

我将使用 **Parallel.ForEachAsync** 来模拟这个，在这里我可以控制 **MaxDegreeOfParallelism** 。

```java
var parallelOptions = new ParallelOptions
{
    MaxDegreeOfParallelism = _maxParallelism,
    CancellationToken = cancellationToken
};

await Parallel.ForEachAsync(
    Enumerable.Range(0, _maxParallelism),
    parallelOptions,
    async (_, token) =>
    {
        await ProcessOutboxMessages(token);
    });
```

我们可以在一分钟内处理 **179,000** 条消息，或者使用五个工作进程达到每秒 **2,983** 条消息的处理速度。

我以为这应该会快得多。到底是怎么回事？

没有并行处理，我们能够达到约 **2700** MPS。

一个新的瓶颈出现了：批量发布消息。

发布时间从约 289 毫秒增加到约 1,540 毫秒。

有趣的是，如果你将基础发布时间（对于一个工作节点）乘以工作节点的数量，你大致可以得到新的发布时间。

我们浪费了大量的时间等待消息代理的确认。

我们该如何解决这个问题？

## 批量发布消息

RabbitMQ 支持批量发布消息。我们可以在配置 MassTransit 时通过调用 **ConfigureBatchPublish** 方法启用此功能。MassTransit 会在将消息发送到 RabbitMQ 之前对其进行缓冲，以提高吞吐量。

```java
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("Queue"), hostCfg =>
        {
            hostCfg.ConfigureBatchPublish(batch =>
            {
                batch.Enabled = true;
            });
        });

        cfg.ConfigureEndpoints(context);
    });
});
```

仅凭这一小改动，让我们用五个工作进程重新运行测试。

这次我们能够在 1 分钟内处理 **1,956,000** 条消息。

这使我们达到了惊人的约 **32,500** MPS。

每天处理超过 **28亿** 条消息。

我可以在这里结束，但还有一件事我想展示给你看。

## 关闭发布者确认（危险操作）

你还可以做一件事（但不推荐），那就是关闭发布者确认。这意味着调用 **Publish** 不会等待消息被代理确认（ack'd）。这可能会导致可靠性问题，甚至可能丢失消息。

话虽如此，在关闭发布者确认的情况下，我确实设法达到了约 **37,000** MPS。

```java
cfg.Host(builder.Configuration.GetConnectionString("Queue"), hostCfg =>
{
    hostCfg.PublisherConfirmation = false; // 危险！不推荐
    hostCfg.ConfigureBatchPublish(batch =>
    {
        batch.Enabled = true;
    });
});
```

## 扩展的关键考虑因素

尽管我们已经实现了令人印象深刻的吞吐量，但在实际系统中实施这些技术时，请考虑以下因素：

1. **消费者容量**：您的消费者能否跟上？提高生产者吞吐量而不匹配消费者容量可能会导致积压。在扩展时请考虑整个管道。
2. **交付保证**：我们的优化保持了至少一次交付。设计消费者以实现幂等性，以处理偶尔的重复消息。
3. **消息顺序**：使用 **FOR UPDATE SKIP LOCKED** 进行并行处理可能会导致消息顺序错乱。对于严格的顺序要求，可以考虑在消费者端使用收件箱模式来缓冲消息。收件箱允许我们即使消息到达顺序不对，也能按正确的顺序处理消息。
4. **可靠性 vs. 性能权衡**：关闭发布者确认可以提高速度，但存在消息丢失的风险。根据您的具体需求，权衡性能与可靠性。

通过解决这些因素，您将创建一个高性能的发件箱处理器，它能与您的系统架构无缝集成。

## 总结

我们已经从最初的发件箱处理器走了很长的路。以下是我们的成就：

1. 优化数据库查询与智能索引
2. 改进的消息发布与批量处理
3. 批量处理以简化数据库更新
4. 扩展发件箱处理，使用并行工作节点
5. 利用 RabbitMQ 的批量发布功能

结果？我们将处理能力从每秒 **1,350** 条消息提升到令人印象深刻的 **32,500** 条消息每秒。这意味着每天超过 **28亿** 条消息！

扩展性不仅仅关乎原始速度，而是要识别并解决每个步骤中的瓶颈。通过测量、优化和重新思考我们的方法，我们实现了巨大的性能提升。

今天就到这里。希望这些内容对你有帮助。

PS. 点击下方【阅读原文】获取源代码
