# [.NET 中使用 EF Core 拆分查询](https://www.nikolatech.net/blogs/ef-core-single-split-query-dotnet)

Entity Framework Core 是一个功能丰富且强大的 ORM，它简化了在.NET 中操作关系型数据库的工作。

然而，要真正发挥其全部潜力，深入了解其底层工作原理至关重要。

开发人员常遇到的一个陷阱是笛卡尔爆炸，当 EF Core 在单个查询中使用联接加载相关实体时，就可能会发生这种情况。

今天，我们将探讨什么是笛卡尔爆炸，它为什么会发生，以及如何避免它以保持查询的性能和效率。

## 笛卡尔爆炸

笛卡尔爆炸是指当组合多个数据集时，导致组合数量呈指数级增长的现象。

这是一个示例：

```csharp
var blog = await dbContext.Blogs
    .Include(b => b.Tags)
    .Include(b => b.Posts)
    .ToListAsync();
```

```sql
SELECT b."Id", b."CreatedAt", t."Id", t."BlogId", t."Name", p."Id", p."BlogId", p."Titles"
FROM "Blogs" AS b
LEFT JOIN "Tags" AS t ON b."Id" = t."BlogId"
LEFT JOIN "Posts" AS p ON b."Id" = p."BlogId"
ORDER BY b."Id", t."Id"
```

在此示例中，Posts 和 Tags 都是 Blog 的集合导航属性，它们在同一级别加载。这会导致为每个博客生成帖子和标签的所有组合。

如果一个博客有 5 个标签和 10 篇文章，那么每个博客的结果将包含 5 × 10 = 50 行。

有了这些重复的行，你的性能会受到影响，内存使用量也会显著增加。

## 拆分查询

幸运的是，我们可以轻松解决这个问题，从 EF Core 5 开始，我们可以创建拆分查询。

拆分查询允许 EF Core 将单个 LINQ 查询分解为多个 SQL 查询，每个包含的集合导航对应一个查询。

这种方法避免了在加载多个集合时由包含大量连接的查询所导致的行重复问题：

```csharp
var blog = await dbContext.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()
    .ToListAsync();
```

只需添加 **AsSplitQuery** 方法，EF Core 就会执行单独的查询来加载 Blogs、Tags 和 Posts，而不是生成一个包含多个连接的大型 SQL 查询：

### 1. 查询博客

```sql
SELECT b."Id", b."CreatedAt"
FROM "Blogs" AS b
ORDER BY b."Id"
```

这将检索主要的 Blogs 实体。

### 2. 查询标签

```sql
SELECT t."Id", t."BlogId", t."Name", b."Id"
FROM "Blogs" AS b
INNER JOIN "Tags" AS t ON b."Id" = t."BlogId"
ORDER BY b."Id"
```

这会检索与每个博客相关联的标签。

### 3. 查询帖子

```sql
SELECT p."Id", p."BlogId", p."Titles", b."Id"
FROM "Blogs" AS b
INNER JOIN "Posts" AS p ON b."Id" = p."BlogId"
ORDER BY b."Id"
```

这会检索与每个博客关联的帖子。

这些结果随后在内存中合并，显著减少了结果集中的冗余数据并提高了性能。

拆分查询在处理多对多或一对多关系时特别有用，因为在这种情况下发生笛卡尔爆炸的可能性很高。

此外，如果你通过 ID 获取博客但它不存在，EF Core 甚至不会发送标签和帖子的查询，这很棒，对吧？

这是因为 EF Core 是顺序发送这些 SQL 语句的，这在高并发系统中可能会出现问题，因为在查询执行之间相关数据可能会发生变化。

此外，由于数据是通过多次往返数据库获取的，网络延迟可能成为瓶颈，特别是在云环境中。

在加载相关实体时，没有一刀切的策略。当收益大于成本时使用拆分查询，并且始终对两种方法进行基准测试。

## 全局配置拆分查询

如果你的应用程序频繁需要拆分查询，你可以将它们配置为默认行为：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(
            @"Server=(localdb)mssqllocaldb;Database=EFQuerying;Trusted_Connection=True;ConnectRetryCount=0",
            o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
}
```

使用此配置，所有包含多个集合导航的查询将默认使用拆分查询。

你仍然可以通过使用 **AsSingleQuery** 方法显式请求单个查询来为特定查询覆盖此行为：

```csharp
var blog = await dbContext.Blogs
    .Include(b => b.Tags)
    .Include(b => b.Posts)
    .AsSingleQuery()
    .ToListAsync();
```

## 总结

笛卡尔爆炸是一个影响重大的问题，当在单个查询中加载多个集合导航时，它可能会降低性能。

幸运的是，拆分查询通过将集合包含分离为独立的查询，提供了一个强大的解决方案，减少了行重复并提高了效率。

也就是说，拆分查询有其自身的权衡，例如增加对数据库的往返调用以及在并发环境中可能出现的一致性问题。

理解这些权衡是关键。

今天就到这里。期待与你再次相见。
