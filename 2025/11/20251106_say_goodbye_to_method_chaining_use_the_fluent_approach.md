# [告别方法链式调用：使用流畅接口方法](https://medium.com/@devesh.akgec/say-goodbye-to-method-chaining-use-the-fluent-approach-bbe26f239f36)

我们将看到如何使用流畅接口模式将传统的顺序方法调用进行转换。

假设我们想要设计一个需要按顺序调用方法的工作流，那么我们将学习如何巧妙地编写代码。

## 理解问题

让我们通过简单的顺序方法调用来理解问题。

考虑下面的 Order 类

```csharp
public class Order
{
    public void AddItems()
    {
        Console.WriteLine(" Items added to order.");
    }

    public void CalculateTotal()
    {
        Console.WriteLine("Total calculated.");
    }

    public void ApplyDiscount()
    {
        Console.WriteLine("Discount applied.");
    }

    public void MakePayment()
    {
        Console.WriteLine(" Payment completed.");
    }

    public void GenerateInvoice()
    {
        Console.WriteLine("Invoice generated.");
    }

    public void Execute()
    {
        AddItems();
        CalculateTotal();
        ApplyDiscount();
        MakePayment();
        GenerateInvoice();
    }
}
```

我们在这里调用 Execute 方法，在其内部所有方法都按顺序调用。

### 上述代码中的问题

- 方法调用过多 — 每个步骤都必须手动调用。
- 难以维护 — 如果步骤顺序发生变化，每个调用点都必须更新。
- 缺乏流程控制 — 可能会意外地按错误顺序调用步骤。

## 解决方案

### 流畅模式

让我们按如下方式修改 Order 类

```csharp
public class OrderV1
{
    public OrderV1 AddItem(string name, decimal price)
    {
        Console.WriteLine($" Added item: {name} - {price:C}");
        return this;
    }

    public OrderV1 ApplyDiscount(decimal percent)
    {
        Console.WriteLine($" Applied discount: {percent}%");
        return this;
    }

    public OrderV1 MakePayment(string method)
    {
        Console.WriteLine($" Payment done via: {method}");
        return this;
    }

    public OrderV1 GenerateInvoice()
    {
        Console.WriteLine(" Invoice generated.");
        return this;
    }
}
```

### 使用示例

```csharp
internal class FluentWorkFlow
{
    public void ProcessOrder()
    {
        new OrderV1()
            .AddItem("Laptop", 1200)
            .ApplyDiscount(10)
            .MakePayment("Credit Card")
            .GenerateInvoice();
    }
}
```

### 解释说明

我们可以看到这是一种非常简洁的方法，让我们来看看它是如何工作的

- 每个方法返回相同的对象（this）
- 你可以将多个方法调用链接在一起
- 每个方法修改状态或执行操作

这非常简单易用。我建议每位开发者在日常编码工作中都应该使用这些方法。

今天就到这里。希望这些内容对你有帮助。
