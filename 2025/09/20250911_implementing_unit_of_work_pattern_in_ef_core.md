# [在 EF Core 中实现工作单元模式](https://antondevtips.com/blog/implementing-unit-of-work-pattern-in-ef-core)

工作单元（UoW）模式是一种设计模式，它允许管理应被视为单一事务的多个操作。今天，我们将探讨工作单元模式以及如何在 Entity Framework Core（EF Core）中实现它。

在 EF Core 中，工作单元模式通常包装 DbContext，提供一个抽象层，通过将多个仓储的操作收集到单个事务中来协调它们的工作。一旦所有操作准备就绪，工作单元会一次性将它们提交到数据库，确保要么所有操作都成功，要么都不成功。这保证了数据的一致性。

工作单元模式在需要同时完成多个操作的场景中特别有用。如果你有多个实体，可能会为每个实体创建一个仓储。即使实体之间相互关联 - 你也不应该创建一个庞大的单一仓储。

工作单元模式可以处理来自多个存储库的数据，确保所有更改一起持久化，避免可能导致数据处于不一致状态的部分更新。

这种模式也具有良好的可扩展性。随着应用程序的增长和存储库数量的增加，工作单元模式通过协调跨存储库的操作来帮助管理复杂性，确保它们都参与同一事务。

**何时使用工作单元模式：**

- **复杂事务**：当你的应用程序对多个实体执行多个操作，而这些操作必须作为单个事务提交时。
- **多个仓储**：如果你的架构涉及多个仓储，工作单元模式可以帮助管理它们之间的协调。
- **撤销或回滚支持**：如果在事务过程中出现问题，需要撤销或回滚更改。

## 在 EF Core 中实现工作单元模式

今天我将向你展示如何为一个负责创建和更新客户、订单以及已订购产品发货信息的应用程序实现工作单元模式。

此应用程序包含以下实体：

- 客户
- 订单，订单项
- 货物，货物项

我正在为我的实体使用领域驱动设计实践。让我们来探索几个实体：

```csharp
public class Customer
{
    public Guid Id { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string Email { get; private set; }
    public string PhoneNumber { get; private set; }
    public IReadOnlyList<Order> Orders => _orders.AsReadOnly();

    private readonly List<Order> _orders = [];

    private Customer() { }

    public static Customer Create(
        string firstName,
        string lastName,
        string email,
        string phoneNumber)
    {
        return new Customer
        {
            Id = Guid.NewGuid(),
            FirstName = firstName,
            LastName = lastName,
            Email = email,
            PhoneNumber = phoneNumber
        };
    }

    public void AddOrder(Order order)
    {
        _orders.Add(order);
    }
}
```

```csharp
public class Order
{
    public Guid Id { get; private set; }

    public string OrderNumber { get; private set; }

    public Guid CustomerId { get; private set; }

    public Customer Customer { get; private set; }

    public DateTime Date { get; private set; }

    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    private readonly List<OrderItem> _items = new();

    private Order() { }

    public static Order Create(string orderNumber, Customer customer, List<OrderItem> items)
    {
        return new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = orderNumber,
            Customer = customer,
            CustomerId = customer.Id,
            Date = DateTime.UtcNow
        }.AddItems(items);
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

    public string Product { get; private set; } = null!;

    public int Quantity { get; private set; }

    public Guid OrderId { get; private set; }

    public Order Order { get; private set; } = null!;

    private OrderItem() { }

    public OrderItem(string productName, int quantity)
    {
        Id = Guid.NewGuid();
        Product = productName;
        Quantity = quantity;
    }
}
```

在我的项目中，我喜欢使用 Clean Architecture 和 Vertical Slices 的组合。我喜欢采用 Clean Architecture 中的 Domain 和 Infrastructure 项目，并将 Application 和 Presentation Layer 合并到 Vertical Slices 中。在某些情况下，我甚至可以将 Infrastructure、Application 和 Presentation 都合并到 Vertical Slices 中。

对于这些实体，我有相应的仓储及其实现：

```csharp
public interface ICustomerRepository
{
    Task AddAsync(Customer customer, CancellationToken cancellationToken);

    Task UpdateAsync(Customer customer, CancellationToken cancellationToken);

    Task<Customer?> GetByIdAsync(Guid customerId, CancellationToken cancellationToken);

    Task<bool> ExistsByEmailAsync(string email, CancellationToken cancellationToken);
}

public interface IOrderRepository
{
    Task AddAsync(Order order, CancellationToken cancellationToken);

    Task UpdateAsync(Order order, CancellationToken cancellationToken);

    Task<Order?> GetByNumberAsync(string orderNumber, CancellationToken cancellationToken);

    Task<bool> ExistsByNumberAsync(string orderNumber, CancellationToken cancellationToken);
}

public interface IShipmentRepository
{
    Task<bool> ExistsByOrderIdAsync(Guid orderId, CancellationToken cancellationToken);

    Task AddAsync(Shipment shipment, CancellationToken cancellationToken);

    Task<Shipment?> GetByNumberAsync(string shipmentNumber, CancellationToken cancellationToken);
}
```

我将 Order 和 OrderItems 归为一个仓储库，同时也将 Shipment 和 ShipmentItem 归为一个仓储库。然而，所有实体都是相互关联的，为它们创建一个庞大的单一仓储库将是一个糟糕的决定。

创建订单时，我们还需要创建相应的发货信息，这两个操作必须是原子性的。如果我们实现两次数据库调用，当订单在数据库中创建成功而发货信息创建失败时，可能会导致数据不一致：

```csharp
await orderRepository.AddAsync(order, cancellationToken);
await shipmentRepository.AddAsync(shipment, cancellationToken);
```

在这种情况下，我们可以使用 IUnitOfWork 来解决我们的一致性问题。让我们来实现它。

首先，我们需要定义 IUnitOfWork 接口：

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

你有两种实现 IUnitOfWork 的方法：

- 创建一个接收 EF Core DbContext 作为构造函数参数的实现
- 或者直接使用 EF Core 的 DbContext，因为它已经开箱即用地实现了 IUnitOfWork 模式

两种方法都可以，我更喜欢使用第二种方法。

我的 ShippingDbContext 继承自 IUnitOfWork 接口：

```csharp
public class ShippingDbContext(DbContextOptions<ShippingDbContext> options)
    : DbContext(options), IUnitOfWork
{
    public DbSet<Shipment> Shipments { get; set; }
    public DbSet<ShipmentItem> ShipmentItems { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<Customer> Customers { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.HasDefaultSchema("shipping");

        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ShippingDbContext).Assembly);
    }
}
```

IUnitOfWork.SaveChangesAsync 的签名与基类中现有的 DbContext.SaveChangesAsync 方法完全相同。因此，你无需在 DbContext 中执行任何其他操作。

你需要注册 IUnitOfWork 接口并从当前作用域解析 ShippingDbContext ：

```csharp
services.AddScoped<IUnitOfWork>(c => c.GetRequiredService<ShippingDbContext>());
```

现在让我们重构所有仓储的 AddAsync 方法，并移除对 DbContext 的 SaveChangesAsync 调用：

```csharp
public async Task AddAsync(Order order, CancellationToken cancellationToken)
{
    await context.Set<Order>().AddAsync(order, cancellationToken);
}
```

保存更改的操作被委托给我们的工作单元，因此我们可以按如下方式更新代码：

```csharp
await orderRepository.AddAsync(order, cancellationToken);
await shipmentRepository.AddAsync(shipment, cancellationToken);
await unitOfWork.SaveChangesAsync(cancellationToken);
```

主要思想是所有仓储都在 EF Core 的变更跟踪器中进行相应的更改，而 UnitOfWork 则将它们全部保存在单个原子事务中。

以下是处理订单创建的完整代码：

```csharp
public async Task<ErrorOr<OrderResponse>> Handle(
    CreateOrderCommand request,
    CancellationToken cancellationToken)
{
    var customer = await customerRepository.GetByIdAsync(request.CustomerId, cancellationToken);
    if (customer is null)
    {
        logger.LogWarning("Customer with ID '{CustomerId}' does not exist", request.CustomerId);
        return Error.NotFound($"Customer with ID '{request.CustomerId}' does not exist");
    }

    var order = Order.Create(
        orderNumber: GenerateNumber(),
        customer,
        request.Items.Select(x => new OrderItem(x.ProductName, x.Quantity)).ToList()
    );

    var shipment = Shipment.Create(
        number: GenerateNumber(),
        orderId: order.Id,
        address: request.ShippingAddress,
        carrier: request.Carrier,
        receiverEmail: request.ReceiverEmail,
        items: []
    );

    var shipmentItems = CreateShipmentItems(order.Items, shipment.Id).ToList();
    shipment.AddItems(shipmentItems);

    await orderRepository.AddAsync(order, cancellationToken);
    await shipmentRepository.AddAsync(shipment, cancellationToken);
    await unitOfWork.SaveChangesAsync(cancellationToken);

    logger.LogInformation("Created order: {@Order} with shipment: {@Shipment}", order, shipment);

    var response = order.MapToResponse();
    return response;
}
```

在我们的应用程序中，可能会遇到这样的用例：当新客户在网站上下订单时，我们需要：

- 创建客户
- 创建包含订单项的订单
- 创建包含运输项目的运输单

使用工作单元模式将变得如下简单：

```csharp
await customerRepository.AddAsync(customer, cancellationToken);
await orderRepository.AddAsync(order, cancellationToken);
await shipmentRepository.AddAsync(shipment, cancellationToken);
await unitOfWork.SaveChangesAsync(cancellationToken);
```

## 总结

工作单元模式是一种强大的设计模式，可以显著提高基于 EF Core 的应用程序的完整性、可维护性和可测试性。通过将一组操作视为单个事务，它确保了数据一致性，并降低了跨多个存储库的事务管理复杂性。现在你应该已经清楚地了解了如何在应用程序中实现 EF Core 的工作单元模式。

今天就到这里。期待与你再次相见。
