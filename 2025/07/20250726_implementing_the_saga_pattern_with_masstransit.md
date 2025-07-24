# [使用 MassTransit 实现 Saga 模式](https://www.milanjovanovic.tech/blog/implementing-the-saga-pattern-with-masstransit)

长运行的业务流程通常涉及多个服务的协同工作。以电子商务订单为例：你需要处理支付、更新库存并通知发货。传统的分布式事务使用两阶段提交（2PC）看似是一个解决方案，但它们带来了显著的缺点。

主要问题是什么？服务不能对其他服务的操作方式或所需时间做出假设。如果支付服务需要人工审批怎么办？如果库存检查延迟了怎么办？在多个服务之间长时间持有数据库锁是不切实际的，可能会导致系统范围的故障。

让我们看看 Saga 模式如何解决这些问题，并使用 MassTransit 来实现它。

## 理解 Saga 模式

Saga 是一系列相关的本地事务，其中每个步骤都有一个定义好的操作，如果出现问题还有一个补偿操作。我们不是使用一个大型的原子事务，而是将过程分解为可管理的步骤进行协调。

这是一个简单的订单处理流程：

每个步骤都是独立的，如果需要可以进行补偿。如果库存服务报告商品缺货，我们可以退还付款。这种方法既灵活又可靠，避免了紧密耦合。

## 使用 MassTransit 实现 Saga 模式

MassTransit 通过与 Automatonymous 集成，提供了一种基于状态机的实现 Saga 模式的方法。理解状态机的工作原理对于实现高效的 Saga 至关重要。

### 状态机基础知识

状态机由几个关键组件组成：

1. **状态**：表示你的 Saga 实例可能的状态
2. **事件**：可以触发状态转换的消息
3. **行为**：在特定状态下接收到事件时发生的操作
4. **实例**：包含特定 Saga 的数据和当前状态

这是我们的订单处理 Saga 的状态机图：

每个状态机自动包含 **Initial** 和 **Final** 状态。 **Initial** 状态是新 Saga 实例开始的地方，而 **Final** 状态标志着 Saga 生命周期的结束。

### 定义 Saga 实例

Saga 实例保存特定流程的数据：

```java
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; }

    // 业务数据
    public decimal OrderTotal { get; set; }
    public string? PaymentIntentId { get; set; }
    public DateTime? OrderDate { get; set; }
    public string? CustomerEmail { get; set; }
}
```

**CorrelationId** 唯一标识了 Saga 实例，而 **CurrentState** 跟踪其当前状态。任何其他属性存储该过程所需的业务数据。

### 定义事件

事件是可以触发状态转换的消息。它们必须与特定的 Saga 实例相关联：

```java
public record OrderSubmitted
{
    public Guid OrderId { get; init; }
    public decimal Total { get; init; }
    public string Email { get; init; }
}

public record PaymentProcessed
{
    public Guid OrderId { get; init; }
    public string PaymentIntentId { get; init; }
}

public record InventoryReserved
{
    public Guid OrderId { get; init; }
}

public record OrderFailed
{
    public Guid OrderId { get; init; }
    public string Reason { get; init; }
}
```

### 构建状态机

让我们将订单处理流程实现为一个状态机：

```java
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public OrderStateMachine()
    {
        Event(() => OrderSubmitted, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => OrderFailed, x => x.CorrelateById(m => m.Message.OrderId));

        InstanceState(x => x.CurrentState);

        Initially(
            When(OrderSubmitted)
                .Then(context =>
                {
                    context.Saga.OrderTotal = context.Message.Total;
                    context.Saga.CustomerEmail = context.Message.Email;
                    context.Saga.OrderDate = DateTime.UtcNow;
                })
                .PublishAsync(context => context.Init<ProcessPayment>(new
                {
                    OrderId = context.Saga.CorrelationId,
                    Amount = context.Saga.OrderTotal
                }))
                .TransitionTo(ProcessingPayment)
        );

        During(ProcessingPayment,
            When(PaymentProcessed)
                .PublishAsync(context => context.Init<ReserveInventory>(new
                {
                    OrderId = context.Saga.CorrelationId
                }))
                .TransitionTo(ReservingInventory),
            When(OrderFailed)
                .TransitionTo(Failed)
                .Finalize()
        );

        During(ReservingInventory,
            When(InventoryReserved)
                .PublishAsync(context => context.Init<OrderConfirmed>(new
                {
                    OrderId = context.Saga.CorrelationId
                }))
                .TransitionTo(Completed)
                .Finalize(),
            When(OrderFailed)
                .PublishAsync(context => context.Init<RefundPayment>(new
                {
                    OrderId = context.Saga.CorrelationId,
                    Amount = context.Saga.OrderTotal
                }))
                .TransitionTo(Failed)
                .Finalize()
        );

        SetCompletedWhenFinalized();
    }

    public State ProcessingPayment { get; private set; }
    public State ReservingInventory { get; private set; }
    public State Completed { get; private set; }
    public State Failed { get; private set; }

    public Event<OrderSubmitted> OrderSubmitted { get; private set; }
    public Event<PaymentProcessed> PaymentProcessed { get; private set; }
    public Event<InventoryReserved> InventoryReserved { get; private set; }
    public Event<OrderFailed> OrderFailed { get; private set; }
}
```

状态机定义了可能的状态和转换。每个步骤在需要时可以触发补偿操作。例如，如果库存预留失败，我们会自动触发支付退款。

### 实现消息消费者

服务通过消费和发布消息与 Saga 进行交互。以下是一个支付处理消费者的示例：

```java
public class ProcessPaymentConsumer(
    IPaymentService paymentService,
    ILogger<ProcessPaymentConsumer> logger) : IConsumer<ProcessPayment>
{
    public async Task Consume(ConsumeContext<ProcessPayment> context)
    {
        try
        {
            var paymentResult = await paymentService.ProcessPaymentAsync(
                context.Message.OrderId,
                context.Message.Amount
            );

            if (paymentResult.Succeeded)
            {
                await context.Publish<PaymentProcessed>(new
                {
                    OrderId = context.Message.OrderId,
                    PaymentIntentId = paymentResult.PaymentIntentId
                });
            }
            else
            {
                await context.Publish<OrderFailed>(new
                {
                    OrderId = context.Message.OrderId,
                    Reason = paymentResult.FailureReason
                });
            }
        }
        catch (Exception ex)
        {
            logger.LogError(
                ex,
                "Failed to process payment for order {OrderId}",
                context.Message.OrderId);

            await context.Publish<OrderFailed>(new
            {
                OrderId = context.Message.OrderId,
                Reason = "Payment processing error"
            });
        }
    }
}
```

每个消费者处理业务流程的特定部分，并通过事件与 Saga 进行通信。这种关注点分离使得每个服务可以专注于其特定的职责，而 Saga 则协调整体流程。

### 使用 PostgreSQL 持久化配置 MassTransit

为了持久化 Saga 状态，我们将使用 PostgreSQL。首先，让我们安装所需的包：

```bash
Install-Package MassTransit.EntityFrameworkCore
Install-Package Npgsql.EntityFrameworkCore.PostgreSQL
```

为 Saga 持久化创建一个 **DbContext** ：

```java
public class OrderSagaDbContext : SagaDbContext
{
    public OrderSagaDbContext(DbContextOptions options) : base(options)
    {
    }

    protected override IEnumerable<ISagaClassMap> Configurations
    {
        get
        {
            yield return new OrderStateMap();
        }
    }
}

public class OrderStateMap : SagaClassMap<OrderState>
{
    protected override void Configure(EntityTypeBuilder<OrderState> entity, ModelBuilder model)
    {
        entity.Property(x => x.CurrentState).HasMaxLength(64);
        entity.Property(x => x.CustomerEmail).HasMaxLength(256);
        entity.Property(x => x.PaymentIntentId).HasMaxLength(64);
    }
}
```

在您的应用程序中配置 MassTransit：

```java
builder.Services.AddDbContext<OrderSagaDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")));

builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Pessimistic;
            r.AddDbContext<DbContext, OrderSagaDbContext>();
            r.UsePostgres();
        });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("RabbitMQ"));
        cfg.ConfigureEndpoints(context);
    });
});
```

## 这种方法的优点

使用 MassTransit 实现 Saga 模式具有多个优点：

1. **容错性**：每个步骤可以独立重试。并且我们可以补偿失败的操作，使我们的系统更具韧性。
2. **状态可见性**：Saga 的状态机提供了对每个流程所处位置的清晰洞察，使得调试和监控变得简单直接。
3. **松散耦合**：服务通过消息进行通信，允许它们独立演化，同时保持流程的完整性。
4. **可维护性**：可以对单个步骤进行更改，而不会影响其他步骤。

状态机方法也使业务流程变得明确。每个状态和转换都定义清晰，使得理解和维护工作流程变得更加容易。

## 要点

使用 MassTransit 实现的 Saga 模式为管理分布式业务流程提供了强大的解决方案。与其处理分布式事务，不如获得清晰的状态管理、自动故障补偿以及在不阻塞资源的情况下处理长时间运行操作的能力。

今天就到这里。期待与你再次相见。

PS. 点击下方【阅读原文】获取使用 MassTransit 实现 Saga 模式的源代码
