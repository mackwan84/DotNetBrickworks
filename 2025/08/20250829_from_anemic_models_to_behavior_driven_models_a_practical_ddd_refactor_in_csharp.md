# [从贫血模型到行为驱动模型：C#中的实用 DDD 重构](https://www.milanjovanovic.tech/blog/from-anemic-models-to-behavior-driven-models-a-practical-ddd-refactor-in-csharp)

如果你曾经处理过遗留的 C#代码库，你就会了解贫血领域模型的痛苦。你可能曾经打开过一个 **OrderService** （与生产代码的相似之处纯属巧合），然后想"这个文件无所不做"。定价逻辑、折扣规则、库存检查、数据库写入——全部都塞进了一个类中。它能运行——直到不能运行为止。新功能变成了回归轮盘赌，测试覆盖率急剧下降，因为领域被埋没在了基础设施之下。

这是贫血领域模型的典型症状，在这种模型中，实体只不过是数据的持有者，所有逻辑都存在于其他地方。这使得系统更难理解，每次变更都变成了一场猜谜游戏。但如果我们能够将行为一次一条规则地推回领域，会怎样呢？

今天，我们将：

- **检查**一个典型的贫血实现。
- **识别**导致系统脆弱的隐藏业务规则。
- 一次**重构**一步，逐步重构出行为丰富的聚合。
- **突出**具体的收益，这样你就能向团队成员证明这一改变的合理性。

所有内容只需 6 分钟阅读，但这种模式可扩展到任何遗留系统。

## 起点：上帝式服务类

下面是一个（不幸地很常见的） **OrderService** 。除了计算总额，它还：

- 应用 5% 的 VIP 折扣，
- 如果任何产品缺货则抛出异常，并且
- 拒绝超过客户信用额度的订单。

```csharp
public void PlaceOrder(Guid customerId, IEnumerable<OrderItemDto> items)
{
    var customer = _db.Customers.Find(customerId);
    if (customer is null)
    {
        throw new ArgumentException("Customer not found");
    }

    var order = new Order { CustomerId = customerId };

    foreach (var dto in items)
    {
        var inventory = _inventoryService.GetStock(dto.ProductId);
        if (inventory < dto.Quantity)
        {
            throw new InvalidOperationException("Item out of stock");
        }

        var price = _pricingService.GetPrice(dto.ProductId);
        var lineTotal = price * dto.Quantity;
        if (customer.IsVip)
        {
            lineTotal *= 0.95m; // 5% 的 VIP 折扣
        }

        order.Items.Add(new OrderItem
        {
            ProductId = dto.ProductId,
            Quantity = dto.Quantity,
            UnitPrice = price,
            LineTotal = lineTotal
        });
    }

    order.Total = order.Items.Sum(i => i.LineTotal);

    if (customer.CreditUsed + order.Total > customer.CreditLimit)
    {
        throw new InvalidOperationException("Credit limit exceeded");
    }

    _db.Orders.Add(order);
    _db.SaveChanges();
}
```

### 这里有什么问题？

- 分散的规则：折扣应用、库存验证和信用额度检查都深藏在服务内部。
- 紧耦合： **OrderService** 必须了解定价、库存和 EF Core 才能下订单。
- 痛苦的测试：每个单元测试都需要为数据库访问、定价、库存以及 VIP 与非 VIP 流程创建模拟对象。

> 目标：将这些规则嵌入到领域内部，以便应用层只负责协调工作。

## 在我们接触代码之前的指导原则

1. 保护不变性应靠近数据。库存、折扣和信用检查应归属于数据所在之处—— **Order** 聚合内部。
2. 暴露意图，隐藏机制。应用层应该读起来像一个故事："下订单"，而不是"计算总额、检查信用、写入数据库"。
3. 分片重构。每一步移动都是安全且可编译的；无需大规模重写。
4. 在纯粹与实用之间取得平衡。只有当收益（清晰度、安全性、可测试性）超过额外的代码行数时，才移动规则。

## 逐步重构

这里的目标不是追求纯粹或学术性的领域驱动设计。而是逐步提高内聚性，为领域表达自身创造空间。

在每一步，我们都会问：这个行为是否应该由领域拥有？如果是，我们就将其向内拉。

### 嵌入创建与验证逻辑

第一步是让聚合体负责构建自身。一个静态 **Create** 方法为我们提供了一个单一入口点，所有不变量都可以在此快速失败。

虽然将库存验证推入 **Order** 中可以提高可测试性，但它确实将订单流程与库存可用性耦合在一起。在某些领域中，你可能会将其建模为领域事件并进行异步验证。

```csharp
public static Order Create(
    Customer customer,
    IEnumerable<(Guid productId, int quantity)> lines,
    IPricingService pricingService,
    IInventoryService inventoryService)
{
    var order = new Order(customer.Id);

    foreach (var (productId, quantity) in lines)
    {
        if (inventoryService.GetStock(productId) < quantity)
        {
            throw new InvalidOperationException("Item out of stock");
        }

        var unitPrice = pricingService.GetPrice(productId);
        order.AddItem(productId, quantity, unitPrice, customer.IsVip);
    }

    order.EnsureCreditWithinLimit(customer);

    return order;
}
```

为什么？现在如果任何不变量被破坏，创建会立即失败。服务不再微观管理库存或折扣。

请注意，我们现在遵循的是"告知，不要询问"原则。服务不再是检查条件然后操作订单，而是告知订单自行创建，并内置必要的验证。这是向封装性转变的根本性转变。

### 将服务注入到领域方法中

将诸如 **IPricingService** 或 **IInventoryService** 之类的服务传递到 **Order.Create** 等域方法中，乍一看可能显得不合常规。但这是一个深思熟虑的设计选择：它将编排保留在领域模型内部，业务逻辑自然属于这里，而不是用程序化工作流使应用服务变得臃肿。

这种方法在保持实体自主性的同时，仍然符合依赖注入原则——依赖项是显式传递的，而不是从内部解析的。这是一种强大的技术，但应该有选择地使用——仅当操作明显符合领域职责范围并且能从直接访问外部服务中受益时。

### 保护聚合的内部状态

```csharp
private readonly List<OrderItem> _items = new();
public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

private void AddItem(Guid productId, int quantity, decimal unitPrice, bool isVip)
{
    if (quantity <= 0)
    {
        throw new ArgumentException("Quantity must be positive");
    }

    var finalPrice = isVip ? unitPrice * 0.95m : unitPrice;
    _items.Add(new OrderItem(productId, quantity, finalPrice));

    RecalculateTotal();
}

private void EnsureCreditWithinLimit(Customer customer)
{
    if (customer.CreditUsed + Total > customer.CreditLimit)
    {
        throw new InvalidOperationException("Credit limit exceeded");
    }
}
```

#### 何必多此一举？

- 封装：消费者不能直接修改 _items ，确保不变量得到保持。
- 自我保护：领域模型保护自身的一致性，而不是依赖于服务级别的检查。
- 真正的面向对象编程：对象现在结合了数据和行為，正如面向对象编程所期望的那样。
- 更简单的服务：应用服务可以专注于协调而非业务规则。

### 将应用层精简为纯编排

```csharp
public void PlaceOrder(Guid customerId, IEnumerable<OrderLineDto> lines)
{
    var customer = _db.Customers.Find(customerId);
    if (customer is null)
    {
        throw new ArgumentException("Customer not found");
    }
    var input = lines.Select(l => (l.ProductId, l.Quantity));

    var order = Order.Create(customer, input, _pricingService, _inventoryService);

    _db.Orders.Add(order);
    _db.SaveChanges();
}
```

**PlaceOrder** 方法从 44 行减少到 14 行，且不包含任何业务逻辑。

## 我们获得的收益

### 在重构之前

- 服务拥有定价、库存、折扣和信用检查功能。
- 单元测试需要大量的 EF Core 和服务模拟。
- 添加新规则意味着需要修改多个文件。

### 重构之后

- 聚合体拥有所有业务规则；服务仅负责编排。
- 纯领域测试 — 无需数据库容器。
- 大多数变更都隔离在 Order 聚合中。

## 总结

重构贫血模型的真正价值不在于技术性，而在于战略性。

通过将业务逻辑移近数据，你可以：

- 减少变更的影响范围
- 使业务规则显式且可测试
- 为验证、事件和不变量等战术模式打开大门

但你不需要进行大规模重写。从一条规则开始。重构它。然后是下一条。

这就是遗留系统演变为可维护架构的方式。

今天就到这里。期待与你再次相见。
