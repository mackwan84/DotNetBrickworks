# [EF 10 中的命名查询过滤器](https://www.milanjovanovic.tech/blog/named-query-filters-in-ef-10-multiple-query-filters-per-entity)

Entity Framework Core 的全局查询过滤器长期以来一直是对实体的所有查询应用通用条件的便捷方式。在诸如软删除和多租户之类的场景中尤其有用，因为你希望相同的 WHERE 子句自动添加到每个查询中。

然而，EF Core 的早期版本存在一个重大限制：每个实体类型只能定义一个过滤器。如果你需要组合多个条件（例如软删除和租户隔离），你要么必须编写显式的 && 表达式，要么在特定查询中手动禁用并重新应用过滤器。

在 EF 10 中，这一情况有所改变。新的命名查询过滤器功能允许你向单个实体附加多个过滤器并按名称引用它们。然后你可以根据需要禁用单个过滤器，而不是一次关闭所有过滤器。

让我们来探索这一新功能、它的重要性以及一些实用的使用方式。

## 什么是查询过滤器？

如果你已经使用 EF Core 一段时间，可能已经熟悉全局查询过滤器。查询过滤器是一个 EF 会自动应用到特定实体类型的所有查询的条件。在内部，EF 在查询该实体时会添加一个 WHERE 子句。典型用途包括：

- **软删除**：过滤掉 IsDeleted 为 true 的行，这样被删除的记录默认不会在查询中出现
- **多租户**：按 TenantId 过滤，使每个租户只能看到自己的数据

例如，软删除过滤器可以这样配置：

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter(order => !order.IsDeleted);
```

在启用该过滤器后，对 Orders 的每个查询都会自动排除软删除的记录。要包含已删除的数据（例如，用于管理员报表），可以在查询上调用 **IgnoreQueryFilters()** 。缺点是该实体上的所有过滤器都会被禁用，这可能导致意外泄露你不打算显示的数据。

## 使用多个查询过滤器

到目前为止，EF 每个实体只允许一个查询过滤器。如果你在同一实体上调用了 HasQueryFilter 两次，第二次调用会覆盖第一次。要合并过滤器，你必须使用 && 编写一个包含所有条件的表达式：

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter(order => !order.IsDeleted && order.TenantId == currentTenantId);
```

这个方法虽然可行，但会导致无法选择性地禁用其中一个条件。 IgnoreQueryFilters() 会同时禁用两者，迫使你手动重新应用仍需要的过滤器。EF 10 引入了更好的替代方案：**命名查询过滤器**。

要将多个筛选器附加到一个实体，请为每个筛选器调用 HasQueryFilter 并为其指定一个名称：

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter("SoftDeletionFilter", order => !order.IsDeleted)
    .HasQueryFilter("TenantFilter", order => order.TenantId == tenantId);
```

在内部，EF 会根据你提供的名称创建单独的过滤器。现在你可以仅关闭软删除过滤器，同时保留租户过滤器：

```csharp
// 返回当前租户下所有（包括软删除的）订单
var allOrders = await context.Orders.IgnoreQueryFilters(["SoftDeletionFilter"]).ToListAsync();
```

如果你省略参数数组， IgnoreQueryFilters() 会禁用该实体的所有过滤器。

## 对筛选器名称使用常量

命名过滤器使用字符串键。在代码库中到处硬编码这些名称很容易引入拼写错误和脆弱的“魔法字符串”。为避免这种情况，请为你的过滤器名称定义常量或枚举，并在需要的地方重复使用它们。例如：

```csharp
public static class OrderFilters
{
    public const string SoftDelete = nameof(SoftDelete);
    public const string Tenant = nameof(Tenant);
}

modelBuilder.Entity<Order>()
    .HasQueryFilter(OrderFilters.SoftDelete, order => !order.IsDeleted)
    .HasQueryFilter(OrderFilters.Tenant, order => order.TenantId == tenantId);

var allOrders = await context.Orders.IgnoreQueryFilters([OrderFilters.SoftDelete]).ToListAsync();
```

将筛选器名称集中定义在一个位置可以减少重复并提高可维护性。另一个最佳实践是将 ignore 调用封装在扩展方法或仓库中，这样使用者就不需要直接与筛选器名称交互。例如：

```csharp
public static IQueryable<Order> IncludeSoftDeleted(this IQueryable<Order> query)
    => query.IgnoreQueryFilters([OrderFilters.SoftDelete]);
```

这使你的意图变得明确，并将筛选逻辑集中到一个地方。

## 总结

在 EF 10 中引入命名查询筛选器消除了 EF 全局查询筛选器功能的长期限制之一。你现在可以：

- 将多个筛选器附加到单个实体并分别管理它们
- 在 LINQ 查询中使用 **IgnoreQueryFilters(["FilterName"])** 有选择性地禁用特定过滤器
- 简化像软删除加多租户这样的常见模式，而无需诉诸复杂的条件逻辑

命名查询过滤器可以成为一种强大工具，保持查询的简洁并封装你的领域逻辑。

无论你是在构建隔离租户数据的 SaaS 应用，还是确保被删除的记录在你明确需要之前保持隐藏，EF 10 的命名查询过滤器都提供了你一直在等待的灵活性。

今天就到这里。期待与你再次相见。
