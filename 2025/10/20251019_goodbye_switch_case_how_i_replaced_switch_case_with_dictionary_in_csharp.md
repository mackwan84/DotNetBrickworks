# [再见 Switch Case：我如何在 C# 中用字典替换 Switch 语句](https://medium.com/@devesh.akgec/goodbye-switch-case-how-i-replaced-switch-case-with-dictionary-in-c-70f21fd6c5b8)

> 从老式 Switch 到 C# 中现代智能代码的字典实现

在我们的实际项目中，我们有很多情况需要使用 switch 语句，而这些语句会随着时间的推移不断增长，变得越来越难以管理。

今天，我们将学习如何巧妙地避免冗长的 switch 语句。

## 理解问题

假设有这么一个场景：

```csharp
using System;

class Program
{
    static void ProcessOrder(string orderType)
    {
        switch (orderType)
        {
            case "NEW_ORDER":
                Console.WriteLine("Processing new order.");
                // 处理新订单的逻辑
                break;

            case "CANCEL_ORDER":
                Console.WriteLine("Cancelling order.");
                // 处理取消订单的逻辑
                break;

            case "SHIPPING":
                Console.WriteLine("Shipping order.");
                // 处理发货的逻辑
                break;

            case "DELIVERED":
                Console.WriteLine("Order delivered.");
                // 处理已交付的逻辑
                break;

            default:
                Console.WriteLine("Unknown order type.");
                break;
        }
    }

    static void Main(string[] args)
    {
        ProcessOrder("NEW_ORDER");
    }
}
```

这么写的问题是：

1. 违反 SOLID 原则
    - 在大型 switch 语句中，很难遵循 SOLID 设计指南中的单一职责原则（SRP）和开闭原则（OCP）
    - 一个类应该只有一个改变的理由（单一职责原则），并且一个类应该对扩展开放，但对修改关闭（开闭原则）。
    - 如果你通过添加新的 case 来修改 switch 语句，你就违反了这两个原则。
2. 难以维护
    - 随着时间的推移，switch 语句会变得越来越长和复杂，难以阅读和维护。
    - 当我们在 switch 语句中添加新情况时，必须更新代码，单个函数中的代码行数会增加，这将使代码难以维护。
3. 测试困难
    - 大量的 switch 语句会使代码更难阅读和测试。switch 块内的每个情况可能包含多行代码，包括业务逻辑、副作用和错误处理。这种复杂性使单元测试变得困难，因为很难隔离与每种情况相关的单独行为。

## 让我们用字典来解决这个问题

我们将用字典替换 switch 语句，其中每个键代表一种订单类型，而值是一个委托或方法，用于处理该订单类型的逻辑。

```csharp
class Program
{
    // 针对订单的每个状态对应一个处理方法

    static void NewOrder()
    {
        Console.WriteLine("Processing new order.");
        // 处理新订单的逻辑
    }

    static void CancelOrder()
    {
        Console.WriteLine("Cancelling order.");
        // 处理取消订单的逻辑
    }

    static void Shipping()
    {
        Console.WriteLine("Shipping order.");
        // 处理发货的逻辑
    }

    static void Delivered()
    {
        Console.WriteLine("Order delivered.");
        // 处理已交付的逻辑
    }

    static void UnknownOrder()
    {
        Console.WriteLine("Unknown order type.");
        // 处理未知订单类型的逻辑
    }  
}
```

首先我们定义了需要在 switch 情况下执行的 Action 方法。

接着为 switch case 定义字典代码。

```csharp
static void Main(string[] args)
{
    // 定义一个字典，将订单类型映射到它们对应的方法
    Dictionary<string, Action> orderActions = new Dictionary<string, Action>
    {
        { "NEW_ORDER", NewOrder },
        { "CANCEL_ORDER", CancelOrder },
        { "SHIPPING", Shipping },
        { "DELIVERED", Delivered }
    };

    // 示例用法
    ProcessOrder(orderActions, "NEW_ORDER");
    ProcessOrder(orderActions, "CANCEL_ORDER");
    ProcessOrder(orderActions, "UNKNOWN_ORDER");

    static void ProcessOrder(Dictionary<string, Action> orderActions, string orderType)
    {
        // 检测订单类型是否存在于字典中并执行相应的操作
        if (orderActions.ContainsKey(orderType))
        {
            orderActions[orderType]();  // 调用执行对应的方法
        }
        else
        {
            UnknownOrder();  // 处理未知订单类型的默认行为
        }
    }
}
```

代码解释：

1. 操作字典：使用 Dictionary\<string, Action\> 将订单类型（字符串键）映射到相应的方法（Action 值）。C# 中的 Action 委托表示不返回值的方法（即 void 方法）。
2. 处理订单：在 ProcessOrder() 方法中，我们使用 ContainsKey() 检查字典中是否存在该订单类型。如果存在，我们调用相应的操作方法。如果订单类型不存在，则调用 UnknownOrder() 方法。

优点：

- 可扩展性：您只需添加新方法并更新字典，就能轻松添加新的订单类型。无需修改复杂的 switch 语句。
- 可读性和可维护性：代码更加有序，每种订单类型都由专门的方法处理。

希望今天的分享对你有帮助。

今天就到这里。期待与你再次相见。
