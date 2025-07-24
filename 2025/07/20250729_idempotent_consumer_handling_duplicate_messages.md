# [幂等消费者 - 处理重复消息](https://www.milanjovanovic.tech/blog/idempotent-consumer-handling-duplicate-messages)

在事件驱动系统中，当一条消息被重试时会发生什么？

这种情况比你想象的更常见。

最坏的情况是消息**被处理两次**，副作用也可能**被多次应用**。

你想让你的银行账户被双重扣费吗？

当然，我会假设答案是“不”。

你可以使用**幂等消费者模式**来解决这个问题。

今天，我将向您展示：

- 幂等消费者模式的工作原理
- 如何实现幂等消费者
- 你需要考虑的权衡

让我们看看为什么**幂等消费者模式**是有价值的。

## 幂等消费者模式的工作原理

幂等消费者模式背后的理念是什么？

> 幂等操作是指，如果使用相同的输入参数多次调用，它不会产生额外的效果。

我们希望避免多次处理同一条消息。

这将需要我们的消息系统提供精确一次的消息传递保证。而在分布式系统中，这是一个非常难以解决的问题。

至少一次的交付保证较为宽松，我们知道可能会发生重试，并且可能会多次接收到相同的消息。

幂等消费者模式与**至少一次**消息传递配合得很好，解决了**重复消息**的问题。

以下是我们在接收到消息时算法的流程：

1. 消息是否已经被处理？
2. 如果是的，则表示是重复消息，无需进行任何操作
3. 如果不然，我们需要处理这条消息
4. 我们也需要存储消息标识符

我们需要为收到的每条消息分配一个**唯一标识符**，并在数据库中**创建一个表**来存储已处理的消息。

然而，我们如何选择实现消息处理和存储已处理消息标识符的方式是很有趣的。

你可以将幂等消费者实现为一个普通消息处理器的装饰器。

我会展示两种实现方式：

- 懒惰幂等消费者 Lazy idempotent consumer
- 急切幂等消费者 Eager idempotent consumer

## 懒惰幂等消费者

懒惰幂等消费者与上述算法中显示的流程相匹配。

懒惰指的是我们如何存储消息标识符以将其标记为已处理。

在正常流程中，我们处理消息并存储消息标识符。

如果消息处理程序抛出异常，我们不会存储消息标识符，消费者可以再次执行。

以下是实现的样例：

```java
public class IdempotentConsumer<T> : IHandleMessages<T>
    where T : IMessage
{
    private readonly IMessageRepository _messageRepository;
    private readonly IHandleMessages<T> _decorated;

    public IdempotentConsumer(
        IMessageRepository messageRepository,
        IHandleMessages<T> decorated)
    {
        _messageRepository = messageRepository;
        _decorated = decorated;
    }

    public async Task Handle(T message)
    {
        if (_messageRepository.IsProcessed(message.Id))
        {
            return;
        }

        await _decorated.Handle(message);

        _messageRepository.Store(message.Id);
    }
}
```

## 急切幂等消费者

急切幂等消费者与懒惰幂等消费者实现略有不同，但最终结果是一样的。

在此版本中，我们急切地将消息标识符存储在数据库中，然后继续处理消息。

如果处理程序抛出异常，我们需要在数据库中执行清理，并删除已急切存储的消息标识符。

否则，我们可能会使系统处于不一致的状态，因为消息从未被正确处理。

以下是实现的样例：

```java
public class IdempotentConsumer<T> : IHandleMessages<T>
    where T : IMessage
{
    private readonly IMessageRepository _messageRepository;
    private readonly IHandleMessages<T> _decorated;

    public IdempotentConsumer(
        IMessageRepository messageRepository,
        IHandleMessages<T> decorated)
    {
        _messageRepository = messageRepository;
        _decorated = decorated;
    }

    public async Task Handle(T message)
    {
        try
        {
            if (_messageRepository.IsProcessed(message.Id))
            {
                return;
            }

            _messageRepository.Store(message.Id);

            await _decorated.Handle(message);
        }
        catch (Exception e)
        {
            _messageRepository.Remove(message.Id);

            throw;
        }
    }
}
```

## 总结

幂等性是软件系统中一个有趣的问题。

一些操作天生就是幂等的，我们不需要幂等消费者模式的开销。

然而，对于那些本质上不具备幂等性的操作，幂等消费者是一个很好的解决方案。

高级算法很简单，你可以在实现中采取两种方法：

- 懒惰存储消息标识符
- 急切存储消息标识符

我更喜欢使用懒惰方法，只有在处理程序成功完成时才将消息标识符存储在数据库中。

更容易理解，并且少了一次对数据库的调用。

今天就到这里。期待与你再次相见。
