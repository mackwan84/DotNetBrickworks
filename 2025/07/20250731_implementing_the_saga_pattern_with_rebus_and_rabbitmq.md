# [使用 Rebus 和 RabbitMQ 实现 Saga 模式](https://www.milanjovanovic.tech/blog/implementing-the-saga-pattern-with-rebus-and-rabbitmq)

在分布式环境中设计长生命周期流程是一个有趣的工程挑战。

解决这个问题的著名模式是 **Saga**。

**Saga** 是一系列本地事务的序列，每个本地事务更新 **Saga** 状态并发布一个消息，触发 **Saga** 的下一步。

Sagas 有两种形式：

- Orchestrated
- Choreographed

使用 **Orchestrated Saga**，有一个中央组件负责编排各个步骤。

在 **Choreographed Saga** 中，各个流程独立工作，但通过事件相互协调。

今天，我将向你展示如何使用 Rebus 库和 RabbitMQ 进行消息传输，创建一个 **Orchestrated Saga**。

闲话少说，立即开干！

## Rebus 配置

Rebus 是一个免费的 让我们安装以下库：.NET“服务总线”，它适用于实现应用程序组件之间的基于异步消息的通信。

让我们安装以下库：

- **Rebus.ServiceProvider** 用于管理 Rebus 实例
- **Rebus.RabbitMq** 用于 RabbitMQ 消息传输
- **Rebus.SqlServer** 用于 SQL Server 状态持久化

```bash
Install-Package Rebus.ServiceProvider -Version 8.4.0
Install-Package Rebus.RabbitMq -Version 8.0.0
Install-Package Rebus.SqlServer -Version 7.3.1
```

在你的 ASP.NET Core 应用程序中，你需要以下配置。

```csharp
services.AddRebus(
    rebus => rebus
        .Routing(r =>
            r.TypeBased().MapAssemblyOf<Program>("newsletter-queue"))
        .Transport(t =>
            t.UseRabbitMq(
                configuration.GetConnectionString("RabbitMq"),
                inputQueueName: "newsletter-queue"))
        .Sagas(s =>
            s.StoreInSqlServer(
                configuration.GetConnectionString("SqlServer"),
                dataTableName: "Sagas",
                indexTableName: "SagaIndexes"))
        .Timeouts(t =>
            t.StoreInSqlServer(
                builder.Configuration.GetConnectionString("SqlServer"),
                tableName: "Timeouts"))
);

services.AutoRegisterHandlersFromAssemblyOf<Program>();
```

拆解各个配置步骤：

- **Routing** - 配置消息按其类型进行路由
- **Transport** - 配置消息传输机制
- **Sagas** - 配置 Saga 持久化存储
- **Timeouts** - 配置超时持久化存储

你还需要指定发送和接收消息的队列名称。

**AutoRegisterHandlersFromAssemblyOf** 将扫描指定的程序集，并自动注册相应的消息处理程序。

## 使用 Rebus 创建 Saga

我们将为新闻简报的入职流程创建一个 Saga。

当用户订阅新闻简报时，我们希望：

- 立即发送欢迎邮件
- 在7天后发送跟进邮件

创建 Saga 的第一步是通过实现 **ISagaData** 来定义数据模型。我们将保持简单，存储 Email 以进行关联，并为我们的 Saga 中的不同步骤添加两个标志。

```csharp
public class NewsletterOnboardingSagaData : ISagaData
{
    public Guid Id { get; set; }
    
    public int Revision { get; set; }

    public string Email { get; set; }

    public bool WelcomeEmailSent { get; set; }

    public bool FollowUpEmailSent { get; set; }
}
```

现在我们可以通过继承自 Saga 类并实现 **CorrelateMessages** 方法来定义 **NewsletterOnboardingSaga** 类。

你还配置了 Saga 如何以 **IAmInitiatedBy** 开始，以及 Saga 处理的各个消息以 **IHandleMessages** 实现。

```csharp
public class NewsletterOnboardingSaga :
    Saga<NewsletterOnboardingSagaData>,
    IAmInitiatedBy<SubscribeToNewsletter>,
    IHandleMessages<WelcomeEmailSent>,
    IHandleMessages<FollowUpEmailSent>
{
    private readonly IBus _bus;

    public NewsletterOnboardingSaga(IBus bus)
    {
        _bus = bus;
    }

    protected override void CorrelateMessages(
        ICorrelationConfig<NewsletterOnboardingSagaData> config)
    {
        config.Correlate<SubscribeToNewsletter>(m => m.Email, d => d.Email);

        config.Correlate<WelcomeEmailSent>(m => m.Email, d => d.Email);

        config.Correlate<FollowUpEmailSent>(m => m.Email, d => d.Email);
    } 
}
```

## 消息类型和命名约定

在 Saga 中，你发送的消息有两种类型：

- **Commands** 命令
- **Events** 事件

命令指示接收组件要做什么。

事件则是通知 Saga 刚刚完成了哪个流程。

## 消息驱动的 Saga 编排

**NewsletterOnboardingSaga** 首先处理 **SubscribeToNewsletter** 命令，并发布一个 **SendWelcomeEmail** 命令。

```csharp
public async Task Handle(SubscribeToNewsletter message)
{
    if (!IsNew)
    {
        return;
    }

    await _bus.Send(new SendWelcomeEmail(message.Email));
}
```

**SendWelcomeEmail** 命令由不同的组件处理，当其完成时会发布一个 **WelcomeEmailSent** 事件。

在 **WelcomeEmailSent** 处理程序中，我们更新 Saga 状态并通过调用 **Defer** 发布一个延迟消息。Rebus 将持久化 **SendFollowUpEmail** 命令，并在超时到期时发布它。

```csharp
public async Task Handle(WelcomeEmailSent message)
{
    Data.WelcomeEmailSent = true;

    await _bus.Defer(TimeSpan.FromDays(7), new SendFollowUpEmail(message.Email));
}
```

最后，处理 **SendFollowUpEmail** 命令，并发布 **FollowUpEmailSent** 事件。

我们再次更新 Saga 状态，并调用 **MarkAsComplete** 来完成 Saga 。

```csharp
public Task Handle(FollowUpEmailSent message)
{
    Data.FollowUpEmailSent = true;

    MarkAsComplete();

    return Task.CompletedTask;
}
```

完成 Saga 将从数据库中删除它。

## 使用 Rebus 处理命令

以下是 **SendWelcomeEmail** 命令处理程序的样子。

```csharp
ublic class SendWelcomeEmailHandler : IHandleMessages<SendWelcomeEmail>
{
    private readonly IEmailService _emailService;
    private readonly IBus _bus;

    public SendWelcomeEmailHandler(IEmailService emailService, IBus bus)
    {
        _emailService = emailService;
        _bus = bus;
    }

    public async Task Handle(SendWelcomeEmail message)
    {
        await _emailService.SendWelcomeEmailAsync(message.Email);

        await _bus.Reply(new WelcomeEmailSent(message.Email));
    }
}
```

需要强调的重要一点是，我们使用 **Reply** 方法来发送回一条消息。这将回复到当前消息中指定的返回地址端点。

## 总结

Sagas 在分布式系统中实现长生命周期流程非常实用。每个业务流程由一个本地事务表示，并发布消息以触发 Saga 中的下一步。

尽管 Saga 模式非常强大，但它们的开发、维护和调试也相当复杂。

我们在本期内容中没有涵盖一些重要话题：

- 错误处理
- 消息重试
- 补偿事务

我认为你自己研究这些会很有趣。

今天就到这里。期待与你再次相见。

PS. 点击下方【阅读原文】获取使用 Rebus 和 RabbitMQ 实现 Saga 模式的源代码
