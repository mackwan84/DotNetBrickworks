# [如何使用领域事件构建松耦合系统](https://www.milanjovanovic.tech/blog/how-to-use-domain-events-to-build-loosely-coupled-systems)

在软件工程中，"耦合"指的是软件系统中不同部分之间相互依赖的程度。如果它们是紧密耦合的，对一个部分的修改可能会影响许多其他部分。但如果它们是松散耦合的，对一个部分的修改不会在系统其余部分引发重大问题。

领域事件是一种领域驱动设计（DDD）的战术模式，我们可以用它来构建松耦合系统。

你可以从领域引发一个领域事件，它代表已发生的事实。系统中的其他组件可以订阅此事件并相应地处理它。

今天，您将学到以下内容：

- 什么是领域事件
- 它们与集成事件有何不同
- 如何实现与触发领域事件
- 如何使用 EF Core 发布领域事件
- 如何使用 MediatR 处理领域事件

我们有很多内容要讲，所以让我们开始吧！

## 什么是领域事件？

事件是过去已经发生的事情。

这是一个事实。

不可更改。

领域事件是在领域中发生的事情，领域的其他部分应该意识到这一点。

领域事件允许你明确地表达副作用，并在领域中提供更好的关注点分离。它们是触发领域内多个聚合之间副作用的理想方式。

确保发布领域事件是事务性的，这是你的责任。你将明白为什么说起来容易做起来难。

![领域事件](https://www.milanjovanovic.tech/blogs/mnw_047/domain_events.png?imwidth=3840)

## 领域事件与集成事件

你可能听说过集成事件，现在你正在好奇它们和领域事件之间有什么区别。

从语义上讲，它们是同一件事：对过去发生事件的表示。

然而，它们的意图是不同的，理解这一点很重要。

领域事件：

- 在单个领域内发布和消费
- 使用内存消息总线发送
- 可以同步或异步处理

集成事件：

- 被其他子系统（微服务、限界上下文）消费
- 通过消息代理和队列发送
- 完全异步处理

所以，如果你在思考应该发布什么类型的事件，请考虑事件的意图以及应该由谁来处理这个事件。

领域事件也可用于生成集成事件，这些事件会离开领域边界。

## 实现领域事件

我实现领域事件的首选方法是创建一个 **IDomainEvent** 抽象并实现 **MediatR** **INotification** 。

好处是你可以使用 **MediatR** 的**发布-订阅支持**将通知发布给一个或多个处理程序。

```csharp
using MediatR;

public interface IDomainEvent : INotification
{
}
```

现在你可以实现一个具体的领域事件。

在设计领域事件时，需要考虑以下几个约束条件：

- 不变性 - 领域事件是事实，应该是不变的
- 胖领域事件 vs 瘦领域事件 - 你需要多少信息？
- 事件命名使用过去时

```csharp
public class CourseCompletedDomainEvent : IDomainEvent
{
    public Guid CourseId { get; init; }
}
```

## 引发领域事件

在你创建领域事件后，你希望从领域中触发它们。

我的方法是创建一个 **Entity** 基类，因为只有实体才允许引发领域事件。你可以通过将 **RaiseDomainEvent** 方法设为 protected 来进一步封装引发领域事件的操作。

我们将领域事件存储在内部集合中，以防止其他人访问它。 **GetDomainEvents** 方法用于获取集合的快照，而 **ClearDomainEvents** 方法用于清除内部集合。

```csharp
public abstract class Entity : IEntity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyList<IDomainEvent> GetDomainEvents()
    {
        return _domainEvents.ToList();
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }

    protected void RaiseDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
}
```

现在你的实体可以继承自 **Entity** 基类并引发领域事件：

```csharp
public class Course : Entity
{
    public Guid Id { get; private set; }

    public CourseStatus Status { get; private set; }

    public DateTime? CompletedOnUtc { get; private set; }

    public void Complete()
    {
        Status = CourseStatus.Completed;
        CompletedOnUtc = DateTime.UtcNow;

        RaiseDomainEvent(new CourseCompletedDomainEvent { CourseId = this.Id });
    }
}
```

剩下的就是发布领域事件。

## 如何使用 EF Core 发布领域事件

使用 EF Core 发布领域事件是一种优雅的解决方案。

由于 EF Core 充当了工作单元，你可以使用它来收集当前事务中的所有领域事件并发布它们。

我不喜欢把事情复杂化，只是简单地重写 **SaveChangesAsync** 方法，在数据库中持久化更改后发布领域事件。但你也可以使用拦截器。

```csharp
public class ApplicationDbContext : DbContext
{
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        // 你该什么时候发布领域事件？
        //
        // 1. 在调用 SaveChangesAsync 之前
        //     - 领域事件是同一事务的一部分
        //     - 立即一致性
        // 2. 在调用 SaveChangesAsync 之后
        //     - 领域事件是一个单独的事务
        //     - 最终一致性
        //     - 处理程序可能会失败

        var result = await base.SaveChangesAsync(cancellationToken);

        await PublishDomainEventsAsync();

        return result;
    }
}
```

在这里您需要做出的最重要决定是何时发布领域事件。

我认为在调用 **SaveChangesAsync** 之后发布是最合理的。换句话说，就是在**保存对数据库的更改之后**。

这带来了一些权衡：

- **最终一致性** - 因为消息是在原始事务处理后进行处理的
- **数据库不一致风险** - 因为处理领域事件可能会失败

最终一致性是我可以接受的，所以我选择做出这种权衡。

然而，引入数据库不一致性的风险是一个重大问题。

您可以使用**发件箱模式**来解决这个问题，在该模式中，您将对数据库的更改和领域事件（作为发箱消息）在单个事务中持久化。现在您有了一个有保证的原子事务，领域事件则通过后台作业进行异步处理。

如果你想知道 **PublishDomainEventsAsync** 方法里面是什么：

```csharp
private async Task PublishDomainEventsAsync()
{
    var domainEvents = ChangeTracker
        .Entries<Entity>()
        .Select(entry => entry.Entity)
        .SelectMany(entity =>
        {
            var domainEvents = entity.GetDomainEvents();

            entity.ClearDomainEvents();

            return domainEvents;
        })
        .ToList();

    foreach (var domainEvent in domainEvents)
    {
        await _publisher.Publish(domainEvent);
    }
}
```

## 如何处理领域事件

在我们目前为止创建的所有基础设施中，我们已经准备好为领域事件实现一个处理器。幸运的是，这是整个过程中最简单的一步。

您只需定义一个实现 **INotificationHandler\<T\>** 的类，并将您的领域事件类型指定为泛型参数。

这是一个 **CourseCompletedDomainEvent** 的处理程序，它接收领域事件并发布 **CourseCompletedIntegrationEvent** 以通知其他系统。

```csharp
public class CourseCompletedDomainEventHandler
    : INotificationHandler<CourseCompletedDomainEvent>
{
    private readonly IBus _bus;

    public CourseCompletedDomainEventHandler(IBus bus)
    {
        _bus = bus;
    }

    public async Task Handle(
        CourseCompletedDomainEvent domainEvent,
        CancellationToken cancellationToken)
    {
        await _bus.Publish(
            new CourseCompletedIntegrationEvent(domainEvent.CourseId),
            cancellationToken);
    }
}
```

## 总结

领域事件可以帮助您构建松耦合系统。您可以使用它们将核心领域逻辑与副作用分离，这些副作用可以异步处理。

实现领域事件无需重新发明轮子，你可以使用 EF Core 和 MediatR 库来构建这个功能。

您需要决定何时发布领域事件。在保存数据库更改之前或之后发布都有其各自的权衡。

我更喜欢在保存数据库更改后发布领域事件，并且我使用 Outbox 模式来添加事务保证。这种方法引入了最终一致性，但它也更可靠。

希望这对你有所帮助。

今天就到这里。期待与你再次相见。
