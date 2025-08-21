# [从贫血领域模型重构到丰富领域模型](https://www.milanjovanovic.tech/blog/refactoring-from-an-anemic-domain-model-to-a-rich-domain-model)

贫血领域模型是一种反模式吗？

贫血领域模型在简单应用中表现出色，但如果你拥有丰富的业务逻辑，它们将难以维护和演进。

你业务逻辑和规则的重要部分最终会分散在整个应用程序中。这降低了内聚性和可重用性，并使添加新功能变得更加困难。

富领域模型试图通过封装尽可能多的业务逻辑来解决这个问题。

但是，你如何设计一个丰富的领域模型呢？

这是一个将业务逻辑移入领域并不断完善领域模型的永无止境的过程。

让我们看看如何从贫血领域模型重构到富领域模型。

## 使用贫血领域模型工作

为了理解使用贫血领域模型的工作方式，我将用一个处理 **SendInvitationCommand** 的例子来说明。

我省略了类及其依赖项，以便我们可以专注于 **Handle** 方法。它从数据库加载一些实体，执行验证，执行业务逻辑，最后将更改持久化到数据库并发送电子邮件。

它已经实现了一些良好的实践，比如使用仓储和返回结果对象。

然而，它正在使用一个贫血领域模型。

有几点表明了这一点：

- 无参数构造函数
- 公共属性设置器
- 暴露的集合

换句话说 - 表示领域实体的类只包含数据属性而没有行为。

贫血领域模型的问题是：

- 操作的可发现性
- 潜在的代码重复
- 缺乏封装

我们将应用一些技术将逻辑下沉到领域层，并尝试使模型更加领域驱动。希望你能看到这将带来的价值和好处。

```csharp
public async Task<Result> Handle(SendInvitationCommand command)
{
    var member = await _memberRepository.GetByIdAsync(command.MemberId);

    var gathering = await _gatheringRepository.GetByIdAsync(command.GatheringId);

    if (member is null || gathering is null)
    {
        return Result.Failure(Error.NullValue);
    }

    if (gathering.Creator.Id == member.Id)
    {
        throw new Exception("Can't send invitation to the creator.");
    }

    if (gathering.ScheduledAtUtc < DateTime.UtcNow)
    {
        throw new Exception("Can't send invitation for the past.");
    }

    var invitation = new Invitation
    {
        Id = Guid.NewGuid(),
        Member = member,
        Gathering = gathering,
        Status = InvitationStatus.Pending,
        CreatedOnUtc = DateTime.UtcNow
    };

    gathering.Invitations.Add(invitation);

    _invitationRepository.Add(invitation);

    await _unitOfWork.SaveChangesAsync();

    await _emailService.SendInvitationSentEmailAsync(member, gathering);

    return Result.Success();
}
```

## 将业务逻辑移入领域

目标是将尽可能多的业务逻辑移入领域层。

让我们从 **Invitation** 实体开始，并为其定义一个构造函数。我可以通过在构造函数内部设置 **Status** 和 **CreatedOnUtc** 属性来简化设计。我还将使其成为 internal ，这样 **Invitation** 实例只能在域内创建。

```csharp
public sealed class Invitation
{
    internal Invitation(Guid id, Gathering gathering, Member member)
    {
        Id = id;
        Member = member;
        Gathering = gathering;
        Status = InvitationStatus.Pending;
        CreatedOnUtc = DateTime.Now;
    }

    // 省略其他成员
}
```

我创建 **Invitation** 构造函数 internal 的原因是为了可以在 **Gathering** 实体上引入一个新方法。我们称之为 **SendInvitation** ，它将负责实例化一个新的 **Invitation** 实例并将其添加到内部集合中。

目前， **Gathering.Invitations** 集合是 public ，这意味着任何人都可以获取引用并修改该集合。

我们不希望允许这种情况，所以我们可以做的是将这个集合封装在一个 private 字段后面。这将管理 _invitations 集合的责任转移到了 **Gathering** 类。

现在 **Gathering** 类的样子如下：

```csharp
public sealed class Gathering
{
    private readonly List<Invitation> _invitations = new();

    // 省略其他成员

    public void SendInvitation(Member member)
    {
        var invitation = new Invitation(Guid.NewGuid(), gathering, member);

        _invitations.Add(invitation);
    }
}
```

## 将验证规则移入领域模型

我们可以做的下一件事是将验证规则移入 **SendInvitation** 方法中，进一步丰富领域模型。

不幸的是，这仍然是一种不良实践，因为在验证失败时抛出了"预期"的异常。如果你想使用异常来强制执行验证规则，至少应该正确地执行，并使用特定的异常而不是通用异常。

但使用结果对象来表达验证错误会更好。

```csharp
public sealed class Gathering
{
    // 省略其他成员

    public void SendInvitation(Member member)
    {
        if (gathering.Creator.Id == member.Id)
        {
            throw new Exception("Can't send invitation to the creator.");
        }

        if (gathering.ScheduledAtUtc < DateTime.UtcNow)
        {
            throw new Exception("Can't send invitation for the past.");
        }

        var invitation = new Invitation(Guid.NewGuid(), gathering, member);

        _invitations.Add(invitation);
    }
}
```

以下是使用结果对象的方式：

```csharp
public sealed class Gathering
{
    // 省略其他成员

    public Result SendInvitation(Member member)
    {
        if (gathering.Creator.Id == member.Id)
        {
            return Result.Failure(DomainErrors.Gathering.InvitingCreator);
        }

        if (gathering.ScheduledAtUtc < DateTime.UtcNow)
        {
            return Result.Failure(DomainErrors.Gathering.AlreadyPassed);
        }

        var invitation = new Invitation(Guid.NewGuid(), gathering, member);

        _invitations.Add(invitation);

        return Result.Success();
    }
}
```

这种方法的好处是我们可以为可能的领域错误引入常量。领域错误目录将作为你领域的文档，并使其更具表现力。

最后，这是到目前为止所有更改后的 **Handle** 方法的样子：

```csharp
public async Task<Result> Handle(SendInvitationCommand command)
{
    var member = await _memberRepository.GetByIdAsync(command.MemberId);

    var gathering = await _gatheringRepository.GetByIdAsync(command.GatheringId);

    if (member is null || gathering is null)
    {
        return Result.Failure(Error.NullValue);
    }

    var result = gathering.SendInvitation(member);

    if (result.IsFailure)
    {
        return Result.Failure(result.Errors);
    }

    await _unitOfWork.SaveChangesAsync();

    await _emailService.SendInvitationSentEmailAsync(member, gathering);

    return Result.Success();
}
```

如果你仔细看一下 **Handle** 方法，你会发现它正在做两件事：

- 将更改持久化到数据库
- 发送电子邮件

这意味着它不是原子性的。

数据库事务有可能完成，但邮件发送却失败。此外，发送邮件会减慢方法的执行速度，可能影响性能。

如何让这个方法具备原子性？

通过在后台发送电子邮件。这对我们的业务逻辑并不重要，因此这样做是安全的。

## 使用领域事件表达副作用

你可以使用领域事件来表达在你的领域中发生了某些事情，这些事情可能对系统中的其他组件来说是有趣的。

我经常使用领域事件来触发后台操作，比如发送通知或电子邮件。

让我们引入一个 **InvitationSentDomainEvent** ：

```csharp
public record InvitationSentDomainEvent(Invitation Invitation) : IDomainEvent;
```

我们将在 **SendInvitation** 方法中触发这个领域事件：

```csharp
public sealed class Gathering
{
    private readonly List<Invitation> _invitations;

    // 省略其他成员

    public Result SendInvitation(Member member)
    {
        if (gathering.Creator.Id == member.Id)
        {
            return Result.Failure(DomainErrors.Gathering.InvitingCreator);
        }

        if (gathering.ScheduledAtUtc < DateTime.UtcNow)
        {
            return Result.Failure(DomainErrors.Gathering.AlreadyPassed);
        }

        var invitation = new Invitation(Guid.NewGuid(), gathering, member);

        _invitations.Add(invitation);

        Raise(new InvitationSentDomainEvent(invitation));

        return Result.Success();
    }
}
```

目标是从 **Handle** 方法中移除负责发送邮件的代码：

```csharp
public async Task<Result> Handle(SendInvitationCommand command)
{
    var member = await _memberRepository.GetByIdAsync(command.MemberId);

    var gathering = await _gatheringRepository.GetByIdAsync(command.GatheringId);

    if (member is null || gathering is null)
    {
        return Result.Failure(Error.NullValue);
    }

    var result = gathering.SendInvitation(member);

    if (result.IsFailure)
    {
        return Result.Failure(result.Errors);
    }

    await _unitOfWork.SaveChangesAsync();

    return Result.Success();
}
```

我们只关心执行业务逻辑并将任何更改持久化到数据库。这些更改的一部分也将是领域事件，系统将在后台发布该事件。

当然，我们需要一个相应的领域事件处理器：

```csharp
public sealed class InvitationSentDomainEventHandler
    : IDomainEventHandler<InvitationSentDomainEvent>
{
    private readonly IEmailService _emailService;

    public InvitationSentDomainEventHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }

    public async Task Handle(InvitationSentDomainEvent domainEvent)
    {
        await _emailService.SendInvitationSentEmailAsync(
            domainEvent.Invitation.Member,
            domainEvent.Invitation.Gathering);
    }
}
```

我们实现了两件事：

- 处理 **SendInvitationCommand** 现在是原子性的
- 邮件在后台发送，如果出现错误可以安全地重试

## 总结

设计一个丰富的领域模型是一个渐进的过程，你可以随时间慢慢演进领域模型。

第一步可能是让你的领域模型更具防御性：

- 使用 internal 关键字隐藏构造函数
- 封装集合访问

好处是你的领域模型将拥有细粒度的公共 API（方法），这些方法作为执行业务逻辑的入口点。

当行为被封装在一个类中而不需要模拟外部依赖时，测试起来很容易。

你可以发布领域事件来通知系统发生了重要的事情，任何感兴趣的组件都可以订阅该领域事件。领域事件允许你开发一个解耦的系统，你可以专注于核心领域逻辑，而不必担心副作用。

然而，这并不意味着每个系统都需要一个丰富的领域模型。

你应该务实，并在复杂度是否值得时做出决定。

今天就到这里。期待与你再次相见。
