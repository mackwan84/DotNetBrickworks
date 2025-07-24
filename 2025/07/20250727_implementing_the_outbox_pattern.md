# [实现发件箱模式](https://www.milanjovanovic.tech/blog/implementing-the-outbox-pattern)

在分布式系统中，我们经常面临保持数据库和外部系统同步的挑战。想象一下，将订单保存到数据库，然后向消息代理发布一条消息。如果任一操作失败，系统最终会处于不一致的状态。

发件箱模式通过将消息发布视为数据库事务的一部分来解决这个问题。我们不是直接发布消息，而是将它们保存到数据库中的发件箱表，确保操作的原子性。然后，一个单独的进程可靠地发布这些消息。

今天，我们将深入探讨在 .NET 中实现此模式，涵盖从设置到扩展的所有内容。

## 为什么我们需要发件箱模式？

事务性发件箱模式解决了分布式系统中的一个常见问题。当你需要同时做两件事时，即保存数据并与外部组件通信时，这个问题就会发生。

考虑以下场景：发送订单确认邮件、通知其他系统有关新客户注册的信息，或在下订单后更新库存水平。每一个场景都涉及本地数据变更，并伴随外部通信或更新。

例如，想象一个需要执行以下任务的微服务：

- 在数据库中保存新订单
- 告知其他系统关于这个新订单的信息

如果这些步骤中的任何一个失败，你的系统可能会陷入不一致的状态。也许订单已经保存，但其他人都不知道。或者每个人都认为有一个新订单，但实际上它并没有在数据库中。

这是一个没有使用发件箱模式（Outbox Pattern）的 **CreateOrderCommandHandler** ：

```java
public class CreateOrderCommandHandler(
    IOrderRepository orderRepository,
    IProductInventoryChecker inventoryChecker,
    IUnitOfWork unitOfWork,
    IEventBus eventBus) : IRequestHandler<CreateOrderCommand, OrderDto>
{
    public async Task<OrderDto> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order(request.CustomerId, request.ProductId, request.Quantity, inventoryChecker);

        await orderRepository.AddAsync(order);

        await unitOfWork.CommitAsync(cancellationToken);

        // 在这里数据库事务已经完成

        await eventBus.Send(new OrderCreatedIntegrationEvent(order.Id));

        return new OrderDto { Id = order.Id, Total = order.Total };
    }
}
```

这段代码存在潜在的一致性问题。在数据库事务提交后，可能会出现两个问题：

1. 应用程序可能在事务提交后但事件发送前崩溃。订单会在数据库中创建，但其他系统不会知道这一点。
2. 事件总线在我们尝试发送事件时可能会宕机或无法访问。这将导致订单创建时未能通知其他系统。

事务性发件箱模式通过确保数据库更新和事件发布被视为单个原子操作，帮助解决了这个问题。

序列图展示了发件箱模式如何解决我们的一致性挑战。我们不是将保存数据和发送消息作为独立的步骤，而是在单个数据库事务中同时保存订单和发件消息。这是一个全有或全无的操作——我们不会陷入不一致的状态。

一个独立的发件箱处理器负责实际的消息发送。它持续检查发件箱表中的未发送消息，并将它们发布到消息队列中。处理器在成功发布后将消息标记为已发送，以防止重复。

这里需要明确的一点是，**发件箱模式保证了至少一次的交付**。发件消息至少会被发送一次，但在发生重试的情况下也可能被发送多次。这意味着我们还需要**确保消息消费者具有幂等性**。

## 实现发件箱模式

首先，让我们创建我们的发件箱表，用于存储消息：

```sql
CREATE TABLE outbox_messages (
    id UUID PRIMARY KEY,
    type VARCHAR(255) NOT NULL,
    content JSONB NOT NULL,
    occurred_on_utc TIMESTAMP WITH TIME ZONE NOT NULL,
    processed_on_utc TIMESTAMP WITH TIME ZONE NULL,
    error TEXT NULL
);

-- 我们可以考虑添加这个索引，因为当我们查询未处理的消息时，它将包含正确的排序顺序，并包含 id、type 和 content 的列。
CREATE INDEX IF NOT EXISTS idx_outbox_messages_unprocessed
ON outbox_messages (occurred_on_utc, processed_on_utc)
INCLUDE (id, type, content)
WHERE processed_on_utc IS NULL;
```

我将使用 PostgreSQL 作为此示例的数据库。请注意 content 列的 **jsonb** 类型。它允许在将来需要时对 JSON 数据进行索引和查询。

现在，让我们创建一个类来表示我们的发件箱条目：

```java
public sealed class OutboxMessage
{
    public Guid Id { get; init; }
    public string Type { get; init; }
    public string Content { get; init; }
    public DateTime OccurredOnUtc { get; init; }
    public DateTime? ProcessedOnUtc { get; init; }
    public string? Error { get; init; }
}
```

我们可以这样向发件箱添加消息：

```java
public async Task AddToOutbox<T>(T message, NpgsqlDataSource dataSource)
{
    var outboxMessage = new OutboxMessage
    {
        Id = Guid.NewGuid(),
        OccurredOnUtc = DateTime.UtcNow,
        Type = typeof(T).FullName, // 我们需要这个属性来实现反序列化
        Content = JsonSerializer.Serialize(message)
    };

    await using var connection = await dataSource.OpenConnectionAsync();
    await connection.ExecuteAsync(
        @"""
        INSERT INTO outbox_messages (id, occurred_on_utc, type, content)
        VALUES (@Id, @OccurredOnUtc, @Type, @Content::jsonb)
        """,
        outboxMessage);
}
```

使用领域事件来表示通知是实现这一目标的一种优雅方法。当领域内发生重要事件时，我们将引发一个领域事件。在完成事务之前，我们可以拾取所有事件并将它们存储为发件箱消息。你可以从工作单元或使用 EF Core 拦截器来完成这一点。

## 处理发件箱消息

发件箱处理器是我们接下来需要的组件。这可以是一个物理上独立的过程，也可以是同一过程中的后台工作线程。

我将使用 **Quartz** 来安排发件箱处理的后台作业。这是一个功能强大的库，对安排周期性作业有极好的支持。

现在，让我们实现 **OutboxProcessorJob** ：

```java
[DisallowConcurrentExecution]
public class OutboxProcessorJob(
    NpgsqlDataSource dataSource,
    IPublishEndpoint publishEndpoint,
    Assembly integrationEventsAssembly) : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        await using var connection = await dataSource.OpenConnectionAsync();
        await using var transaction = await connection.BeginTransactionAsync();

        // 你可以将此限制作为参数，以控制批量大小。
        // 我们也可以只选择 id、type 和 content 列。
        var messages = await connection.QueryAsync<OutboxMessage>(
            @"""
            SELECT id AS Id, type AS Type, content AS Content
            FROM outbox_messages
            WHERE processed_on_utc IS NULL
            ORDER BY occurred_on_utc LIMIT 100
            """,
            transaction: transaction);

        foreach (var message in messages)
        {
            try
            {
                var messageType = integrationEventsAssembly.GetType(message.Type);
                var deserializedMessage = JsonSerializer.Deserialize(message.Content, messageType);

                // 我们可以使用重试来提高可靠性。
                await publishEndpoint.Publish(deserializedMessage);

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
                // 我们也可以记录错误。

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

        await transaction.CommitAsync();
    }
}

```

这种方法使用轮询定期从数据库中获取未处理的消息。轮询可能会增加数据库的负载，因为我们需要频繁查询未处理的消息。

处理发件箱消息的另一种方法是使用事务日志尾随。我们可以通过 Postgres 逻辑复制来实现这一点。数据库将从预写日志（WAL）中将变更流式传输到我们的应用程序，我们将处理这些消息并将它们发布到消息代理。你可以使用这种方法来实现基于推送的发件箱处理器。

## 考虑因素与权衡

发件箱模式虽然有效，但引入了额外的复杂性和数据库写入。在高吞吐量系统中，监控其性能至关重要，以确保其不会成为瓶颈。

我建议在发件箱处理器中实现重试机制以提高可靠性。考虑对暂时性故障使用指数退避策略，并对持续性问题的使用断路器，以防止系统在故障期间过载。

实现幂等消息消费者至关重要。网络问题或处理器重启可能导致同一消息多次投递，因此你的消费者必须能够安全地处理重复处理。

随着时间的推移，发件箱表可能会显著增长，从而可能影响数据库性能。尽早实施归档策略非常重要。考虑将已处理的消息移动到冷存储，或在设定的时间后删除它们。

## 扩展发件箱处理

随着系统的增长，你可能会发现单个发件箱处理器无法跟上消息量的增长。这可能导致事件发生与消费者处理之间的延迟增加。

一种直接的方法是增加发件箱处理器作业的频率。你应该考虑每隔几秒运行一次。这样可以显著减少消息处理的延迟。

另一个有效的策略是在获取未处理消息时增加批量大小。通过每次运行处理更多消息，可以提高吞吐量。然而，要小心不要将批量设置得过大，以免导致长时间运行的事务。

对于高容量系统，并行处理发件箱可以非常有效。实现一个锁定机制来认领消息批次，允许多个处理器同时工作而不会发生冲突。你可以使用 **SELECT ... FOR UPDATE SKIP LOCKED** 来认领一批消息。这种方法可以显著提高你的处理能力。

## 总结

发件箱模式是维护分布式系统中数据一致性的强大工具。通过将数据库操作与消息发布解耦，发件箱模式确保即使在出现故障的情况下，你的系统也能保持可靠。

记得保持消费者的幂等性，实施适当的扩展策略，并管理好你的发件箱表的增长。

虽然它增加了一些复杂性，但确保消息传递的好处使其在许多场景中成为一个有价值的模式。

今天就到这里。期待与你再次相见。
