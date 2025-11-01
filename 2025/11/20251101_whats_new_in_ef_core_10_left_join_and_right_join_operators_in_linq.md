# [EF Core 10 新特性：LINQ 中的 LeftJoin 和 RightJoin 操作符](https://www.milanjovanovic.tech/blog/whats-new-in-ef-core-10-leftjoin-and-rightjoin-operators-in-linq)

如果你曾经使用过数据库，你就会了解 LEFT JOIN （以及相应的 RIGHT JOIN ）。这是我们经常使用的东西，即使不是一直使用。但在 Entity Framework Core 中，执行左连接一直...嗯，有点麻烦。

我喜欢那些读起来就像它们所做事情的连接。不幸的是，直到现在，LINQ 还没有一种直接表达左/右连接的方法。你必须通过 GroupJoin 和 DefaultIfEmpty 来绕圈子，这使得代码更难阅读和维护。

但是 .NET 10 终于通过全新的 LeftJoin 和 RightJoin 方法解决了这个问题。

## 什么是 LEFT JOIN ?

LEFT JOIN 返回左侧表的所有行以及右侧表中匹配的行。如果没有匹配项，则右侧为 null。我们使用它的原因：即使"所有者"没有相关行也要保留它们（例如，显示所有产品，即使有些产品没有评论）。

![LEFT JOIN](https://www.milanjovanovic.tech/blogs/mnw_166/left_join.gif?imwidth=2048)

## 旧方法（ GroupJoin + DefaultIfEmpty ）

在 .NET 10 之前，LINQ 中的左连接需要使用组连接（ GroupJoin ），然后通过 DefaultIfEmpty 来保留没有匹配项的左侧行。这种方法虽然可行，但其意图被淹没在复杂的代码中。

有两种编写方式：查询语法和方法语法。

### 查询语法

```csharp
var query =
    from product in dbContext.Products
    join review in dbContext.Reviews on product.Id equals review.ProductId into reviewGroup
    from subReview in reviewGroup.DefaultIfEmpty()
    orderby product.Id, subReview.Id
    select new
    {
        ProductId = product.Id,
        product.Name,
        product.Price,
        ReviewId = (int?)subReview.Id ?? 0,
        Rating = (int?)subReview.Rating ?? 0,
        Comment = subReview.Comment ?? "N/A"
    };
```

以下是 EF Core 为上述查询生成的 SQL：

```sql
SELECT
    p."Id" AS "ProductId",
    p."Name",
    p."Price",
    COALESCE(r."Id", 0) AS "ReviewId",
    COALESCE(r."Rating", 0) AS "Rating",
    COALESCE(r."Comment", 'N/A') AS "Comment"
FROM "Products" AS p
LEFT JOIN "Reviews" AS r ON p."Id" = r."ProductId"
ORDER BY p."Id", COALESCE(r."Id", 0)
```

### 方法语法

```csharp
var query = dbContext.Products
    .GroupJoin(
        dbContext.Reviews,
        product => product.Id,
        review => review.ProductId,
        (product, reviewList) => new { product, subgroup = reviewList })
    .SelectMany(
        joinedSet => joinedSet.subgroup.DefaultIfEmpty(),
        (joinedSet, review) => new
        {
            ProductId = joinedSet.product.Id,
            joinedSet.product.Name,
            joinedSet.product.Price,
            ReviewId = (int?)review!.Id ?? 0,
            Rating = (int?)review!.Rating ?? 0,
            Comment = review!.Comment ?? "N/A"
        })
    .OrderBy(result => result.ProductId)
    .ThenBy(result => result.ReviewId);
```

为什么这样有效： GroupJoin 匹配行，当不存在匹配时， DefaultIfEmpty 插入单个默认值（null），因此左行仍然会出现。然后我们使用 SelectMany 进行扁平化处理。

我想我们都同意，对于像左连接这样常见的操作，这种写法实在是太冗长了。

## EF 10 中的新方法： LeftJoin

现在我们可以直接表达我们的意图。 LeftJoin 是一流的 LINQ，EF Core 会将其转换为 SQL 中的 LEFT JOIN。

```csharp
var query = dbContext.Products
    .LeftJoin(
        dbContext.Reviews,
        product => product.Id,
        review => review.ProductId,
        (product, review) => new
        {
            ProductId = product.Id,
            product.Name,
            product.Price,
            ReviewId = (int?)review.Id ?? 0,
            Rating = (int?)review.Rating ?? 0,
            Comment = review.Comment ?? "N/A"
        })
    .OrderBy(x => x.ProductId)
    .ThenBy(x => x.ReviewId);
```

生成的 SQL 与前面的示例相同。

为什么这样更好：

- 意图清晰：看到 LeftJoin ，你就知道会发生什么。
- 代码更少，可移动部件更少（没有 GroupJoin ，没有 DefaultIfEmpty ，没有 SelectMany ）。
- 结果相同：保留所有产品，评论可能为空。

## 另外新增： RightJoin

RightJoin 保留右侧的所有行，只保留左侧的匹配行。EF Core 将其转换为 RIGHT JOIN。当"必须保留"的一侧是第二个序列时，这非常方便。

概念上：

```csharp
var query = dbContext.Reviews
    .RightJoin(
        dbContext.Products,
        review => review.ProductId,
        product => product.Id,
        (review, product) => new
        {
            ProductId = product.Id,
            product.Name,
            product.Price,
            ReviewId = (int?)review.Id ?? 0,
            Rating = (int?)review.Rating ?? 0,
            Comment = review.Comment ?? "N/A"
        });
```

为什么使用 RightJoin ：当你的报表从 Reviews（保留所有记录）开始，并引入匹配的 Products（如果存在的话）。

这是生成的 SQL：

```sql
SELECT
    p."Id" AS "ProductId",
    p."Name",
    p."Price",
    COALESCE(r."Id", 0) AS "ReviewId",
    COALESCE(r."Rating", 0) AS "Rating",
    COALESCE(r."Comment", 'N/A') AS "Comment"
FROM "Reviews" AS r
RIGHT JOIN "Products" AS p ON r."ProductId" = p."Id"
```

## 总结

想想看，你多久需要一次左连接。显示所有用户及其可选的个人资料设置。所有产品及其可选的评价。所有订单及其可选的配送信息。无处不在！

以前，开发者有时会跳过正确的左连接，而是执行两个单独的查询。或者更糟糕的是，他们会使用内连接而丢失数据。现在没有借口了——它就像任何其他 LINQ 方法一样简单。

编写 LINQ 查询时的几个快速提示：

- 在投影中，保护可空端： review.Comment ?? "N/A"
- 保持投影精简，避免拉取不必要的列
- 在连接键上添加索引以获得更好的查询计划

就是这样。有了 LeftJoin 和 RightJoin ，代码终于与思维模型相匹配。

今天就到这里。希望这些内容对你有帮助。
