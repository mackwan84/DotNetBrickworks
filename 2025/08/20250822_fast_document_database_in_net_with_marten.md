# [使用 Marten 在 .NET 中实现快速文档数据库](https://www.milanjovanovic.tech/blog/fast-document-database-in-net-with-marten)

你知道可以将 **PostgreSQL** 转变为一个功能完备的文档数据库吗？

**Marten** 是一个 .NET 库，它允许开发人员将 **PostgreSQL** 数据库既作为文档数据库，又作为功能齐全的事件存储使用。

除了 Nuget 包外，你不需要安装任何其他东西就能将 **PostgreSQL** 用作文档数据库。**Marten** 依赖于 **PostgreSQL** 9.4 以来可用的 JSONB 支持。

今天，我想向你介绍使用 Marten 的基础知识，并向你展示开始使用它是多么容易。

让我们开始吧。

## 安装与配置 Marten

要开始使用 **PostgreSQL** 作为文档数据库，你需要准备什么？

当然，除了一个正在运行的 **PostgreSQL** 实例外，你还需要安装 **Marten** Nuget 包：

```bash
dotnet add package Marten
```

**Marten** 可以即时构建所需的数据库架构和必要表，我建议在开发中使用这种方法。对于生产环境，你肯定希望通过迁移脚本自行应用架构更改。

要使用依赖注入注册 **Marten**，你需要调用 **AddMarten** 方法。

以下是在 .NET 应用程序中的 **Marten** 配置示例：

```csharp
builder.Services.AddMarten(options =>
{
    options.Connection(builder.Configuration.GetConnectionString("Marten"));
});
```

这将在依赖注入中注册几个服务：

- **IDocumentStore** - 用于创建会话、生成架构迁移和执行批量插入
- **IDocumentSession** - 用于读写操作
- **IQuerySession** - 用于执行查询操作

接下里让我们看看如何使用 **Marten** 来处理文档。

## 使用 Marten 存储文档

在数据库中存储文档非常简单。你需要创建一个新的 **DocumentStore** 实例，并打开一个 **IDocumentSession**，它提供了存储和持久化文档的方法。

让我们看看如何存储一个 Product 文档：

```csharp
var store = DocumentStore.For("Connection string to PostgreSQL");

using var session = store.OpenSession();

var product = new Product
{
    Name = "C# and .NET - Modern Cross-Platform Fundamentals",
    Price = 46.87
};

session.Store(product);

await session.SaveChangesAsync();
```

我们正在创建一个新的 **DocumentStore** 实例，用它来打开到 **PostgreSQL** 的会话。然后我们只需调用 **Store** 并传入 Product 实例。请注意，**Marten** 此时会填充 Product.Id 。Marten 可以填充 Guid 、 int 、 long 和其他数据类型的键。对于数字键，它使用 HiLo 算法。最后，当我们调用 **SaveChangesAsync** 时， Product 会被序列化为 JSON 并作为文档持久化存储。

需要注意的重要一点是，由 **OpenSession** 创建的 **IDocumentSession** 不会自动跟踪实体上的更改。你需要通过在 **DocumentStore** 上调用 **DirtyTrackedSession** 来创建一个启用了脏检查的会话，以启用自动更改检测。

## 使用 Marten 查询文档

**Marten** 对数据库中的文档查询提供了丰富的支持。你可以使用 **LINQ** 编写和执行查询，如果你使用过 **EF Core**，应该对此很熟悉。你也可以编写和执行 SQL 查询，因为它底层仍然是一个 **PostgreSQL** 数据库。

以下是一个示例查询，用于返回价格高于指定值的产品：

```csharp
ar store = DocumentStore.For("Connection string to PostgreSQL");

using var session = store.QuerySession();

var products = session.Query<Product>().Where(p => p.Price > 9.99).ToList();
```

**Marten** 还支持：

- 包含相关文档
- 批量查询
- 分页查询
- 全文搜索

## Marten 的高级选项

**Marten** 可以利用 **PostgreSQL** 提供的全部功能，特别是事务和索引。**Marten** 会话默认是事务性的，要么所有文档一起持久化，要么都不持久化。而且你可以为文档配置索引以加快查询速度。

**Marten** 不仅仅是构建在 **PostgreSQL** 之上的文档数据库！

**Marten** 为你提供了完整的事件溯源支持以及投影功能。这使其成为实现 CQRS 的理想选择。

## 总结

我对 **Marten** 及其所提供的功能感到非常惊叹。**PostgreSQL** 也是我最喜欢的数据库，所以这就像是天作之合。我通常不会对学习新技术感到过于兴奋，但在过去的一周里，**Marten** 一直是我无尽的快乐源泉。

在我考虑将其用于生产环境之前，我还需要探索几个更多的话题：

- 架构迁移
- 关系和外键
- 高级配置选项

考虑到 **PostgreSQL** 比大多数文档数据库更便宜，我认为使用 **Marten** 是一个有趣的选择。而且如果你熟悉 SQL 数据库，你仍然可以运用所有这些知识。

今天就到这里。期待与你再次相见。
