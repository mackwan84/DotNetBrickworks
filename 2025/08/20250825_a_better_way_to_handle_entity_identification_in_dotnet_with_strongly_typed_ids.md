# [.NET 中处理实体识别的更好方法 Strongly Typed ID](https://antondevtips.com/blog/a-better-way-to-handle-entity-identification-in-dotnet-with-strongly-typed-ids)

强类型 ID 是用于在应用程序中表示实体标识符（ID）的自定义类型，而不是使用像 int、Guid 或 string 这样的基本类型。与其直接使用这些基本类型来表示 ID，你可以创建一个封装 ID 值的特定类或结构体。这种方法有助于使代码更具表现力、更安全且更易于维护。

## 基本类型痴迷问题

昨天中，我们探讨了什么是原始类型痴迷，以及如何通过使用值对象来解决这个问题。

简而言之，基本类型痴迷是一种倾向于使用基本数据类型来表示更复杂概念的现象。当基本数据类型（如 int、string 或 DateTime）被过度用于表示领域中的复杂概念时，就会出现基本类型痴迷。

这是一种常见的反模式，可能导致代码不清晰且系统难以维护。

如果你的项目中有多个实体，标识符（ID）也存在同样的问题。如果你对所有实体标识符都使用 Guid ，比如 CustomerId、OrderId、OrderItemId，可能会导致错误地将 OrderId 传递给需要 OrderItemId 的地方。这都是因为在所有地方都使用了相同的类型。

这种基本类型痴迷问题可以通过使用值对象来解决。值对象表示你领域中一个没有身份但由其属性定义的值。

实体标识符的基本类型痴迷问题可以通过使用强类型 ID 来解决，强类型 ID 是一种值对象，但仅应用于实体标识符。

## 实体标识符的基本类型痴迷示例

让我们来探索一个包含 Order 和 OrderItem 实体的应用程序示例。这两个实体都使用 guid 作为它们的实体标识符类型：

```csharp
public class Order
{
    public Guid Id { get; set; }
}

public class OrderItem
{
    public Guid Id { get; set; }
}
```

想象一下，你正在调用 ProcessOrderAsync 方法：

```csharp
public Task ProcessOrderAsync(Guid orderId, Guid orderItemId)
{
    // 处理订单以及相关订单项的逻辑
}

// 正确传参
ProcessOrder(order.Id, orderItem.Id);

// 错误传参 - 编译时没有报错
ProcessOrder(orderItem.Id, order.Id);
```

这段代码编译并成功执行。你注意到问题了吗？

由于我们对 Order.Id 和 OrderItem.Id 都使用 Guid 类型，你可能会以错误的顺序传递参数。最终会导致数据库中出现错误数据，这可能引发严重问题。

这个问题的解决方案是使用强类型 ID，它将实体标识符封装成一个有意义的自定义单元。

## 什么是强类型 ID？

强类型 ID 是用于在应用程序中表示实体标识符（ID）的自定义类型，而不是使用像 int、Guid 或 string 这样的基本类型。与其直接使用这些基本类型来表示 ID，你可以创建一个封装 ID 值的特定类或结构体。这种方法有助于使代码更具表现力、更安全且更易于维护。

### 强类型 ID 的关键特征

- 不可变性：一旦创建，强类型 ID 就不能被更改。任何修改都会产生一个新实例。
- 相等性：强类型 ID 是基于其值进行比较，而不是通过引用。

### 强类型 ID 的优势

- **增强类型安全性**：强类型 ID 最重要的优势之一是增强类型安全性。这减少了意外混用不同类型 ID 的可能性，从而减少 bug 和运行时错误。
- **提高代码清晰度**：强类型 ID 使你的代码更具表现力和自文档性。当你看到接受 OrderId 的方法时，可以立即明确期望的是哪种 ID，而不是通用的 Guid 或 int。
- **更好的领域建模**：在领域驱动设计（DDD）中，强类型 ID 有助于强化实体及其身份的概念。这使得领域模型更加健壮，并与它所代表的真实世界概念更加一致。
- **支持未来增强**：如果 ID 的需求发生变化（例如，需要更改实体标识符的类型），强类型 ID 类可以被更新或扩展以支持这些新需求，而不会破坏现有代码。
- **更轻松的重构**：强类型 ID 使重构更加容易，因为与 ID 相关的逻辑变更可以在一个地方完成，从而降低了在此过程中引入错误的可能性。

现在让我们来探讨在 .NET 中创建强类型 ID 有哪些选项。

## 一个示例应用程序

今天我将向你展示如何为货运应用程序实现强类型 ID，该应用程序负责为订购的产品创建和更新客户、订单和货运。

此应用程序包含以下实体：

- 客户
- 订单、订单项
- 货物、货物项

我正在为我的实体使用领域驱动设计实践。让我们来探索使用原始类型作为实体标识符的 Order 和 Order Item 实体：

```csharp
public class Order
{
    public Guid Id { get; private set; }

    public OrderNumber OrderNumber { get; private set; }

    public Guid CustomerId { get; private set; }

    public Customer Customer { get; private set; }

    public DateTime Date { get; private set; }

    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private readonly List<OrderItem> _items = new();

    private Order() { }

    public static Order Create(OrderNumber orderNumber, Customer customer, List<OrderItem> items)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = orderNumber,
            Customer = customer,
            CustomerId = customer.Id,
            Date = DateTime.UtcNow
        };

        order.AddItems(items);

        return order;
    }

    private Order AddItems(List<OrderItem> items)
    {
        _items.AddRange(items);
        return this;
    }
}
```

```csharp
public class OrderItem
{
    public Guid Id { get; private set; }

    public ProductName Product { get; private set; }

    public int Quantity { get; private set; }

    public Guid OrderId { get; private set; }

    public Order Order { get; private set; } = null!;

    private OrderItem() { }

    public OrderItem(ProductName productName, int quantity)
    {
        Id = Guid.NewGuid();
        Product = productName;
        Quantity = quantity;
    }
}
```

如你所见，两个实体都使用值对象作为它们的属性。如果你想了解更多关于值对象的内容——请务必查看我相应的文章。

现在让我们来探索如何使用强类型 ID 来替换这些用于实体标识符的原始类型。

## 在 .NET 中创建强类型 ID

我可以列出以下几种最流行的创建强类型 ID 的选项：

- 使用 StronglyTypedId 包
- 使用 record 记录类型
- 使用 record struct 记录结构体

让我们更深入地探讨每个选项。

## 使用 StronglyTypedId 库创建强类型 ID

通过使用 StronglyTypedId 库，你可以为你的实体生成强类型 ID 结构体。

首先，你需要安装这个库：

```bash
dotnet add package StronglyTypedId
```

你需要定义一个 partial struct 并添加一个 StronglyTypedId 属性：

```csharp
[StronglyTypedId]
public readonly partial struct OrderId { }

[StronglyTypedId]
public readonly partial struct OrderItemId { }
```

StronglyTypedId 库将使用源生成器来实现 OrderId 和 OrderItemId 结构体。默认情况下，基础类型是 Guid 。

以下是创建 OrderId 的方法：

```csharp
var orderId = new OrderId(Guid.NewGuid());
var orderItemId = new OrderItemId(Guid.NewGuid());
```

你可以使用 Value 属性来检索隐藏在强类型 ID 中的值：

```csharp
var guid = orderId.Value;
var guid2 = orderItemId.Value;
```

如果需要更改类型，请在属性中指定一个 Template ：

```csharp
[StronglyTypedId(Template.Int)]
public readonly partial struct OrderId { }
```

当前支持的内置基础类型包括：

- Guid 默认类型
- int
- long
- string

## 使用记录类型创建强类型 ID

你不必使用外部包，因为 C# 记录已经为你提供了强类型 ID 所需的一切。

记录类型是在 .NET 中创建强类型 ID 的一种非常现代的方式。

记录是不可变的引用类型，它们开箱即用地支持相等性比较。它们的比较是基于属性而非引用。

以下是使用记录类型定义相同强类型 ID 的方法：

```csharp
public record OrderId(Guid Value);

public record OrderItemId(Guid Value);
```

## 使用记录结构创建强类型 ID

记录类型是强类型 ID 的绝佳选择，但它们是引用类型。如果你关心内存分配，可以使用 readonly record struct 作为强类型 ID。它们的行为与 record 相同，但它们是值类型，不会在堆上分配。

这是我创建强类型 ID 的个人选择。

以下是如何使用 record struct 定义强类型 ID 的方法：

```csharp
public readonly record struct OrderId(Guid Value);

public readonly record struct OrderItemId(Guid Value);
```

以下是使用强类型 ID 后的 Order 和 OrderItem 实体的外观：

```csharp
public class Order
{
    public OrderId Id { get; private set; }

    public OrderNumber OrderNumber { get; private set; }

    public CustomerId CustomerId { get; private set; }

    public Customer Customer { get; private set; }

    public DateTime Date { get; private set; }

    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private readonly List<OrderItem> _items = new();

    private Order() { }

    public static Order Create(OrderNumber orderNumber, Customer customer, List<OrderItem> items)
    {
        var order = new Order
        {
            Id = new OrderId(Guid.NewGuid()),
            OrderNumber = orderNumber,
            Customer = customer,
            CustomerId = customer.Id,
            Date = DateTime.UtcNow
        };

        order.AddItems(items);

        return order;
    }

    private Order AddItems(List<OrderItem> items)
    {
        _items.AddRange(items);
        return this;
    }
}
```

```csharp
public class OrderItem
{
    public OrderItemId Id { get; private set; }

    public ProductName Product { get; private set; }

    public int Quantity { get; private set; }

    public OrderId OrderId { get; private set; }

    public Order Order { get; private set; } = null!;

    private OrderItem() { }

    public OrderItem(ProductName productName, int quantity)
    {
        Id = new OrderItemId(Guid.NewGuid());
        Product = productName;
        Quantity = quantity;
    }
}
```

如果你尝试误用实体标识符，将会得到一个编译错误：

![编译错误](https://antondevtips.com/media/code_screenshots/architecture/strongly-typed-ids/img_1.png)

## 在 EF Core 中映射强类型 ID

在实体模型中引入强类型 ID 后，你需要修改 EF Core 映射。

你需要使用转换来告诉 EF Core 如何将强类型 ID 映射到数据库，以及如何将数据库值映射到 ID。

例如，对于 Order 实体：

```csharp
builder.Property(x => x.Id)
    .HasConversion(
        id => id.Value,
        value => new OrderId(value)
    )
    .IsRequired();
```

## 强类型 ID 与请求/响应/DTO 模型

强类型 ID 是你特定于领域的模型，外部世界不应该了解它们。而且，你的公共请求/响应/DTO 模型应该尽可能简单。

在你的请求/响应/DTO 模型中使用简单原始类型，并将它们映射到领域实体，反之亦然，是一种良好的实践。

例如，在我的"创建订单"用例中，我将 CustomerId 与 Guid 类型映射到 CustomerId ：

```csharp
public sealed record CreateOrderRequest(
    Guid CustomerId,
    List<OrderItemRequest> Items,
    Address ShippingAddress,
    string Carrier,
    string ReceiverEmail);

public static CreateOrderCommand MapToCommand(this CreateOrderRequest request)
{
    return new CreateOrderCommand(
        CustomerId: new CustomerId(request.CustomerId),
        Items: request.Items,
        ShippingAddress: request.ShippingAddress,
        Carrier: request.Carrier,
        ReceiverEmail: request.ReceiverEmail
    );
}
```

以及从强类型 OrderId 到 CustomerResponse 的反向映射：

```csharp
public static OrderResponse MapToResponse(this Order order)
{
    return new OrderResponse(
        OrderId: order.Id.Value,
        OrderNumber: order.OrderNumber.Value,
        OrderDate: order.Date,
        Items: order.Items.Select(x => new OrderItemResponse(
            ProductName: x.Product.Value,
            Quantity: x.Quantity)).ToList()
    );
}
```

## 总结

强类型 ID 是提高 .NET 应用程序类型安全性、清晰度和可维护性的强大工具。通过将 ID 封装在专用的记录或结构中，你可以防止常见错误并使代码更具表现力。

C# 记录类型和只读记录结构体为你提供了一种优雅、简单且快速的方式来实现强类型 ID，而无需样板代码。

今天就到这里。期待与你再次相见。
