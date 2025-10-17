# [为什么事件驱动设计是可扩展 .NET 微服务的秘密武器](https://medium.com/itnext/why-event-driven-design-is-the-secret-weapon-for-scalable-net-microservices-0b0c47137612)

> 用 C# 观察者模式和消息队列替代脆弱的 REST 调用，构建解耦、高性能的系统。

掌握 .NET 中的事件驱动架构以构建可扩展的微服务。学习 C# 观察者模式、RabbitMQ 和 Kafka 等消息队列，以及通过代码示例了解解耦策略。

## 为什么这么做

扩展 .NET 微服务不应该意味着为每个新功能重写一半的代码库 —— 然而通过 REST 调用产生的紧密耦合往往恰恰导致了这种情况。

想一想。你添加一个简单的"订单确认后发送折扣邮件"功能，部署它，然后你的支付服务突然崩溃，因为通知服务耗时太长。这就是同步链的隐藏代价。

到 2025 年，随着 .NET 9 的云原生推动以及微服务为从金融科技到医疗保健的各个领域提供支持，事件驱动架构已不再是可选项。它是解耦服务、吸收变化和无混乱扩展的最清晰方式。

今天的分享，我们将涵盖：

- 为什么直接调用在扩展规模时会失败。
- C# 观察者模式作为概念基础。
- 消息队列如何在分布式系统中扩展它。
- MassTransit 实现的分步指南。
- 比较、权衡以及何时不使用事件。
- 用于生产就绪系统的工具、监控和最佳实践。

闲话少说，让我们开始吧。

## .NET 中紧耦合的缺点

### 为什么直接调用无法应对规模化挑战

REST 和 gRPC 非常适合同步 API —— 但它们会引入紧密的依赖关系：

- 订单 → 支付 → 通知 → 日志记录
- 每个服务都必须知道要调用谁、如何调用它们，以及在它们失败时该怎么做。
- 一个瓶颈会级联影响到整个链路。

这种设计在刚开始时感觉很自然。但一旦扩展到 10 个以上的服务，你基本上就是在用 API 玩叠叠乐——一步走错，整个塔就会倒塌。

### 电商示例：打破链条

想象一下：

- 客户下订单。
- 订单服务保存订单信息。
- 调用支付服务 → 扣除费用。
- 调用库存服务 → 预留库存。
- 调用通知服务 → 发送电子邮件。

现在，业务方要求你添加分析功能：实时跟踪订单频率。

接下来会发生什么？你需要再次修改订单服务。

每个新的依赖都会使你的代码膨胀，并减慢部署速度。这就是"微服务"如何变回"分布式单体"的过程。

## 在 C# 中实现观察者模式

观察者模式是事件驱动设计的概念根源。它将主题（发布者）与观察者（订阅者）解耦。

### C# 事件处理程序代码

```csharp
public class Order 
{ 
    public Guid Id { get; set; } 
}

public class OrderService
{
    // 通知订阅者的事件
    public event Action<Order>? OrderPlaced;
    public void PlaceOrder(Order order)
    {
        // 保存订单信息
        Console.WriteLine($"Order {order.Id} saved.");
        OrderPlaced?.Invoke(order); // 通知所有订阅者
    }
}
// 观察者
void SendEmail(Order order) => Console.WriteLine($"Email sent for {order.Id}");
void UpdateInventory(Order order) => Console.WriteLine($"Stock updated for {order.Id}");
// 如何调用
var orderService = new OrderService();
orderService.OrderPlaced += SendEmail;
orderService.OrderPlaced += UpdateInventory;
orderService.PlaceOrder(new Order { Id = Guid.NewGuid() });
```

输出：

```bash
Order 123 saved.
Email sent for 123
Stock updated for 123
```

仅通过一个事件，多个订阅者做出了反应，而订单服务无需知道它们是谁。

### 常见陷阱与解决方案

- 作用域 → 仅在进程内有效。
- 持久性 → 如果应用程序崩溃，事件会丢失。
- 内存泄漏 → 忘记取消订阅会导致引用被保留。
- 注意：在长时间运行的应用程序中，务必取消订阅或使用弱事件模式。

## 扩展到分布式系统的消息队列

观察者模式一旦离开单个进程就会失效。这时消息队列就派上用场了。

### 队列如何实现解耦

像 RabbitMQ、Azure Service Bus 和 Kafka 这样的消息代理将观察者语义扩展到分布式系统：

- 生产者只需发布一次。
- 代理负责处理消息传递、重试和持久化。
- 消费者独立订阅。
- 系统可水平扩展。

### 示例：订单流程

1. 订单服务发布 OrderPlaced 。
2. 支付服务、库存服务和通知服务各自订阅。
3. 稍后添加分析服务？只需订阅即可。无需更改订单服务。

## .NET 集成 Brokers

- RabbitMQ → 轻量级，简单，久经考验。
- Azure Service Bus → 云原生，企业级。
- Kafka → 高吞吐量，非常适合事件流和事件溯源。

注意：不要过早选择 Kafka。对于大多数微服务来说，RabbitMQ 是最佳选择。

## 实际实现：使用 MassTransit 在 .NET 中处理事件

理论已经足够 — 让我们开始实际操作。

### 步骤 1：定义事件契约

```csharp
public record OrderPlacedEvent(Guid OrderId, DateTime OccurredAtUtc);
```

### 步骤 2：发布事件

```csharp
using MassTransit;

public class OrderService
{
    private readonly IPublishEndpoint _publish;
    public OrderService(IPublishEndpoint publish) => _publish = publish;

    public async Task PlaceOrderAsync(Order order, CancellationToken ct = default)
    {
        // 保存订单信息
        Console.WriteLine($"Order {order.Id} saved.");
        await _publish.Publish(new OrderPlacedEvent(order.Id, DateTime.UtcNow), ct);
    }
}
```

### 步骤 3：在另一个服务中消费

```csharp
using MassTransit;
public class SendEmailOnOrderPlaced : IConsumer<OrderPlacedEvent>
{
    public async Task Consume(ConsumeContext<OrderPlacedEvent> ctx)
    {
        var (orderId, when) = ctx.Message;
        Console.WriteLine($"[Email Service] Sending email for order {orderId}");
        await Task.CompletedTask;
    }
}
```

### 步骤 4：使用 RabbitMQ 配置 MassTransit

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<SendEmailOnOrderPlaced>();
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(context);
    });
});
```

除此之外，我们还可以集成 OpenTelemetry 以跨服务追踪事件，并在数据看板中跟踪链接发布/消费跨度。

## 方法比较：REST 与 事件

| 方法 | 优点 | 缺点 |
|------|------|------|
| REST 调用 | 简单直观；易于调试 | 紧耦合，缺乏灵活性 |
| 观察者模式 | 解耦；易于扩展 | 仅限进程内；无持久性 |
| 消息队列 | 解耦；持久性；可扩展 | 复杂性增加；引入额外组件 |

![性能比较](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*m9Z8MHjYhCNo99nmRQzdRQ.png)

![不同消息队列之间的比较](https://miro.medium.com/v2/resize:fit:4000/format:webp/0*kvXHL7c8vHiguax0.png)

## 何时采用事件驱动架构

理想场景

- 与不断发展的消费者的微服务集成。
- 异步工作流程，如订单处理、ETL 和物联网。
- 实时通知和分析。
- 用于审计或商业智能的分析管道。

何时避免使用

- 小型 CRUD 应用（开销不值得）。
- 需要严格一致性的系统。
- 运维能力有限的团队（消息代理 = 运维负担）。

## 必备的 .NET 工具和监控

关键库

- MassTransit → 面向 .NET 的成熟服务总线库。
- CAP → 支持 RabbitMQ、Kafka、SQL Server 的事件总线抽象。
- Scrutor → 用于事件处理程序的灵活依赖注入扫描。

可观测性最佳实践

- Serilog → 结构化日志，将日志视为事件。
- App.Metrics → 监控吞吐量和消费者健康状态。
- OpenTelemetry → 事件流的端到端追踪。

注意事项：务必确保消费者具有幂等性。重试可能导致重复投递。

## 总结

- .NET 中的事件驱动架构解耦服务并吸收变化。
- C# 观察者模式是你的概念基础。
- .NET 中的消息队列实现了持久性、重试和水平扩展。
- 混合系统（REST + 事件）在速度和韧性之间提供了最佳平衡。

你在使用事件驱动的 .NET 系统时遇到的最大挑战是什么——扩展队列还是处理故障？欢迎在评论中分享你的经验和问题。

今天就到这里。期待与你再次相见。
