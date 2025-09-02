# [EF Core 中的规约模式：无需存储库的灵活数据访问](https://antondevtips.com/blog/specification-pattern-in-ef-core-flexible-data-access-without-repositories)

随着 .NET 项目的增长，数据处理变得越来越复杂。许多团队从仓储模式开始，将他们的 EF Core 查询封装在其中。

起初，这工作得很好。但随着项目的发展，你的仓储要么做得不够，要么做得太多。随着业务需求的变化，你的代码变得更难理解和修改。

每次你需要新的筛选器或查询时，你都会添加另一个方法，甚至是一个新的仓储。

你如何解决这个问题？答案是**规约模式**。

在这篇文章中，你将学习**规约模式**在实际 .NET 项目中如何工作，以及为什么它比编写大量仓储来读取数据更好。

在本文中，我们将探讨：

- 为什么仓储模式在实际项目中会成为瓶颈
- 什么是规约模式？
- 如何在 EF Core 中实现规约模式
- 高级规约

闲话少说，让我们开始吧！

## 为什么仓储模式在实际项目中会成为瓶颈

当你的应用程序规模较小时，使用仓储模式似乎很简单。

你将所有数据查询都放在一个地方，例如 **PostRepository** 或 **UserRepository** 。每个方法都旨在回答一个特定问题，例如“获取所有最新帖子”或“通过电子邮件查找用户”。

但随着项目的发展，你会发现一些大问题：

### 1. 仓储变得过于庞大

每个新的业务需求都意味着向仓储添加另一个方法。随着时间的推移，你最终会得到充满类似方法的类：

```csharp
public class PostRepository
{
    public Task<List<Post>> GetPostsByUser(int userId) { ... }
    public Task<List<Post>> GetPopularPosts() { ... }
    public Task<List<Post>> GetPostsByCategory(string category) { ... }
    public Task<List<Post>> GetRecentViralPosts(int daysBack) { ... }
    // 还有很多很多其他方法
}
```

要找到正确的方法，甚至记住每个存储库中已有的内容都变得更加困难。

### 2. 方法命名与重复

你开始编写笨拙冗长的方法名来描述每个筛选器。例如， **GetPostsByCategoryAndLikesCountAndDate** 。

如果需要相同的方法，但按日期降序排序呢？这会导致大量的代码重复。

### 3. 仓储类过多

有时为了保持存储库的精简，需要将其拆分成许多小类。但这样一来，你就失去了将所有内容集中在一处的优势。

现在，要找到必要的仓储并在仓储之间重用逻辑变得更加困难。

请记住，EF Core 的 DbContext 已经实现了存储库和工作单元模式，这在官方 DbContext 的代码摘要中有所说明。当我们在 EF Core 之上创建存储库时，我们是在一个抽象之上创建另一个抽象，这会导致过度设计的解决方案。

那么，现在让我们来探讨一下什么是**规约模式**，以及它如何解决我们的问题。

## 什么是规约模式

**规约模式**是一种使用称为“规约”的小型可重用类来描述你希望从数据库中获取哪些数据的方法。

每个规约都代表一个可以应用于查询的过滤器或规则。这使你可以通过组合简单易懂的类来构建复杂的查询。

规约模式带来了以下好处：

- 可重用性：你可以编写一次规约，并在项目的任何地方使用它。
- 可测试性：规约是基于 EF Core（或任何其他 ORM）的类，因此你可以用单元测试来覆盖它们，或者更好——用集成测试。
- 关注点分离：你的查询逻辑与数据访问代码是分离的。这使得代码保持整洁。

无需在仓储中编写数十个方法，你可以随着需求增长创建新的规约。然后，你可以将这些规约传递给你的 DbContext（如果你仍想使用仓储，也可以传递给它）。

这是一个规约示例，它返回社交媒体应用程序中至少有 150 个赞的病毒式帖子：

```csharp
public class ViralPostSpecification : Specification<Post>
{
    public ViralPostSpecification(int minLikesCount = 150)
    {
        AddFilteringQuery(post => post.Likes.Count >= minLikesCount);
        AddOrderByDescendingQuery(post => post.Likes.Count);
    }
}
```

你可以在代码中的任何地方重用此规约，以获取“病毒式传播”的帖子。

接下来我们看看如何实现规约模式。

## 如何在 EF Core 中实现规约模式

要在 EF Core 中实现规约模式，你需要遵循以下步骤：

### 第一步：定义规约接口

你需要一种通用的方式来描述任何实体的筛选、包含和排序。这是一个简单的接口：

```csharp
public interface ISpecification<TEntity>
    where TEntity : class
{
    Expression<Func<TEntity, bool>>? FilterQuery { get; }
    IReadOnlyCollection<Expression<Func<TEntity, object>>>? IncludeQueries { get; }
    IReadOnlyCollection<Expression<Func<TEntity, object>>>? OrderByQueries { get; }
    IReadOnlyCollection<Expression<Func<TEntity, object>>>? OrderByDescendingQueries { get; }
}
```

### 第二步：创建基础规约类

这个基类包含了构建规约的逻辑。你可以在一个地方添加筛选器、包含项和排序表达式：

```csharp
using System.Linq.Expressions;

public abstract class Specification<TEntity> : ISpecification<TEntity>
    where TEntity : class
{
    private List<Expression<Func<TEntity, object>>>? _includeQueries;
    private List<Expression<Func<TEntity, object>>>? _orderByQueries;
    private List<Expression<Func<TEntity, object>>>? _orderByDescendingQueries;

    public Expression<Func<TEntity, bool>>? FilterQuery { get; private set; }
    public IReadOnlyCollection<Expression<Func<TEntity, object>>>? IncludeQueries => _includeQueries;
    public IReadOnlyCollection<Expression<Func<TEntity, object>>>? OrderByQueries => _orderByQueries;
    public IReadOnlyCollection<Expression<Func<TEntity, object>>>? OrderByDescendingQueries => _orderByDescendingQueries;

    protected Specification() {}

    protected Specification(Expression<Func<TEntity, bool>> query)
    {
        FilterQuery = query;
    }

    protected Specification(ISpecification<TEntity> specification)
    {
        FilterQuery = specification.FilterQuery;

        _includeQueries = specification.IncludeQueries?.ToList();
        _orderByQueries = specification.OrderByQueries?.ToList();
        _orderByDescendingQueries = specification.OrderByDescendingQueries?.ToList();
    }

    protected void AddFilteringQuery(Expression<Func<TEntity, bool>> query)
    {
        FilterQuery = query;
    }

    protected void AddIncludeQuery(Expression<Func<TEntity, object>> query)
    {
        _includeQueries ??= new();
        _includeQueries.Add(query);
    }

    protected void AddOrderByQuery(Expression<Func<TEntity, object>> query)
    {
        _orderByQueries ??= new();
        _orderByQueries.Add(query);
    }

    protected void AddOrderByDescendingQuery(Expression<Func<TEntity, object>> query)
    {
        _orderByDescendingQueries ??= new();
        _orderByDescendingQueries.Add(query);
    }
}
```

以下是每个选项的描述：

- **FilterQuery**：用于测试每个实体是否满足条件的筛选函数
- **IncludeQueries**：描述包含实体的函数集合
- **OrderByQueries** / **OrderByDescendingQueries**：描述如何按升序/降序排列实体的函数

### 第三步：在 EF Core 中应用规约

现在，让我们将 Specification 类连接到 EF Core：

```csharp
public class EfCoreSpecification<TEntity> : Specification<TEntity>
    where TEntity : class
{
    public EfCoreSpecification(ISpecification<TEntity> specification)
        : base(specification) { }
    
    public virtual IQueryable<TEntity> Apply(IQueryable<TEntity> queryable)
    {
        if (FilterQuery is not null)
        {
            queryable = queryable.Where(FilterQuery);
        }

        if (IncludeQueries?.Count > 0)
        {
            queryable = IncludeQueries.Aggregate(queryable,
                (current, includeQuery) => current.Include(includeQuery));
        }

        if (OrderByQueries?.Count > 0)
        {
            var orderedQueryable = queryable.OrderBy(OrderByQueries.First());
            
            orderedQueryable = OrderByQueries.Skip(1)
                .Aggregate(orderedQueryable, (current, orderQuery) => current.ThenBy(orderQuery));
            
            queryable = orderedQueryable;
        }
        
        if (OrderByDescendingQueries?.Count > 0)
        {
            var orderedQueryable = queryable.OrderByDescending(OrderByDescendingQueries.First());
            
            orderedQueryable = OrderByDescendingQueries.Skip(1)
                .Aggregate(orderedQueryable, (current, orderQuery) => current.ThenByDescending(orderQuery));
            
            queryable = orderedQueryable;
        }

        return queryable;
    }
}
```

Apply 方法接受一个数据库查询，并逐步向其添加不同类型的筛选和排序。

首先，它检查是否存在筛选条件（例如“只显示点赞数超过 10 的帖子”），并使用 Where 方法应用该条件。

接下来，它会查找需要一起加载的任何相关数据（称为“包含”），并使用 Include 方法（EF Core 预加载）添加这些数据。

然后它处理排序——它将第一个排序条件与 OrderBy 应用，并将任何额外的排序条件与 ThenBy 应用。

最后，该方法返回应用了所有这些条件后的修改查询，随时可以发送到数据库。

### 第四步：在你的端点中直接使用规约

通过这种模式，你可以将查询逻辑放入名为“规约”的小型可复用类中。你可以直接在 EF Core 中将它们与 DbContext 结合使用。这意味着你根本不需要仓储模式。

现在你可以编写简单、整洁的端点了：

```csharp
public class GetViralPostsEndpoint : IEndpoint
{
    public void MapEndpoint(WebApplication app)
    {
        app.MapGet("/api/social-media/viral-posts", Handle);
    }

    private static async Task<IResult> Handle(
        [FromQuery] int? minLikesCount,
        [FromServices] ApplicationDbContext dbContext,
        [FromServices] ILogger<GetViralPostsEndpoint> logger,
        CancellationToken cancellationToken)
    {
        var specification = new ViralPostSpecification(minLikesCount ?? 150);

        var response = await dbContext
            .ApplySpecification(specification)
            .Select(post => post.ToDto())
            .ToListAsync(cancellationToken);

        logger.LogInformation("Retrieved {Count} viral posts with minimum {MinLikes} likes",
            response.Count, minLikesCount ?? 150);

        return Results.Ok(response);
    }
}
```

我通过一个小的扩展方法将 Specification 应用到 DbContext 上：

```csharp
public static class SpecificationExtensions
{
    public static IQueryable<TEntity> ApplySpecification<TEntity>(
        this ApplicationDbContext dbContext,
        ISpecification<TEntity> specification) where TEntity : class
    {
        var efCoreSpecification = new EfCoreSpecification<TEntity>(specification);

        var query = dbContext.Set<TEntity>().AsNoTracking();
        query = efCoreSpecification.Apply(query);

        return query;
    }
}
```

如果你仍想使用仓储模式，你可以具体化规约查询，并从仓储中返回结果：

```csharp
public abstract class BaseRepository<TEntity> where TEntity : class
{
    private readonly ApplicationDbContext _dbContext;

    public Repository(ApplicationDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<List<TEntity>> WhereAsync(
        ISpecification<TEntity> specification,
        CancellationToken cancellationToken = default)
    {
        var efCoreSpecification = new EfCoreSpecification<TEntity>(specification);

        var query = _dbContext.Set<TEntity>().AsNoTracking();
        query = efCoreSpecification.Apply(query);

        return await query.ToListAsync(cancellationToken).ConfigureAwait(false);
    }
}
```

## 高级规约

规范模式最出色的优点之一是，通过组合小而集中的规范，可以轻松构建高级查询。

你可以使用逻辑运算符（如 AND 和 OR）将两个或多个规约连接起来。

以下是创建 **AndSpecification** 和 **OrSpecification** 的方法：

```csharp
public class AndSpecification<TEntity> : Specification<TEntity>
    where TEntity : class
{
    public AndSpecification(Specification<TEntity> left, Specification<TEntity> right)
    {
        RegisterFilteringQuery(left, right);
    }

    private void RegisterFilteringQuery(Specification<TEntity> left, Specification<TEntity> right)
    {
        var leftExpression = left.FilterQuery;
        var rightExpression = right.FilterQuery;

        if (leftExpression is null && rightExpression is null)
        {
            return;
        }
        
        if (leftExpression is not null && rightExpression is null)
        {
            AddFilteringQuery(leftExpression);
            return;
        }
        
        if (leftExpression is null && rightExpression is not null)
        {
            AddFilteringQuery(rightExpression);
            return;
        }
        
        var replaceVisitor = new ReplaceExpressionVisitor(rightExpression!.Parameters.Single(), leftExpression!.Parameters.Single());
        var replacedBody = replaceVisitor.Visit(rightExpression.Body);

        var andExpression = Expression.AndAlso(leftExpression.Body, replacedBody);
        var lambda = Expression.Lambda<Func<TEntity, bool>>(andExpression, leftExpression.Parameters.Single());

        AddFilteringQuery(lambda);
    }
}
```

```csharp
public class OrSpecification<TEntity> : Specification<TEntity>
    where TEntity : class
{
    public OrSpecification(Specification<TEntity> left, Specification<TEntity> right)
    {
        RegisterFilteringQuery(left, right);
    }

    private void RegisterFilteringQuery(Specification<TEntity> left, Specification<TEntity> right)
    {
        var leftExpression = left.FilterQuery;
        var rightExpression = right.FilterQuery;

        if (leftExpression is null && rightExpression is null)
        {
            return;
        }
        
        if (leftExpression is not null && rightExpression is null)
        {
            AddFilteringQuery(leftExpression);
            return;
        }
        
        if (leftExpression is null && rightExpression is not null)
        {
            AddFilteringQuery(rightExpression);
            return;
        }
        
        var replaceVisitor = new ReplaceExpressionVisitor(
            rightExpression!.Parameters.Single(),
            leftExpression!.Parameters.Single()
        );
        
        var replacedBody = replaceVisitor.Visit(rightExpression.Body);

        var andExpression = Expression.OrElse(leftExpression.Body, replacedBody);
        
        var lambda = Expression.Lambda<Func<TEntity, bool>>(
            andExpression, leftExpression.Parameters.Single());

        AddFilteringQuery(lambda);
    }
}
```

要组合两个查询，我们需要使用 Expression.AndAlso 或 Expression.OrElse 以及表达式访问器：

```csharp
internal class ReplaceExpressionVisitor : ExpressionVisitor
{
    private readonly Expression _oldValue;
    private readonly Expression _newValue;

    /// <summary>
    /// 初始化实例
    /// </summary>
    /// <param name="oldValue">要替换的旧表达式</param>
    /// <param name="newValue">替换旧表达式的新表达式</param>
    public ReplaceExpressionVisitor(Expression oldValue, Expression newValue)
    {
        _oldValue = oldValue;
        _newValue = newValue;
    }

    public override Expression Visit(Expression? node)
        => (node == _oldValue ? _newValue : base.Visit(node))!;
}
```

为了简化这些规范的用法，让我们在 Specification 类中创建 2 个辅助方法：

```csharp
public Specification<TEntity> And(Specification<TEntity> specification)
    => new AndSpecification<TEntity>(this, specification);

public Specification<TEntity> Or(Specification<TEntity> specification)
    => new OrSpecification<TEntity>(this, specification);
```

现在我们来看一个实际的例子。

假设你有一个用于搜索给定类别中帖子的规约：

```csharp
public class PostByCategorySpecification : Specification<Post>
{
    public PostByCategorySpecification(string categoryName)
    {
        AddFilteringQuery(post => post.Category.Name == categoryName);
    }
}
```

你可以组合规范来选择属于“.NET”或“架构”类别的帖子：

```csharp
public class DotNetAndArchitecturePostSpecification : Specification<Post>
{
    public DotNetAndArchitecturePostSpecification()
    {
        var dotNetSpec = new PostByCategorySpecification(".NET");
        var architectureSpec = new PostByCategorySpecification("Architecture");

        // 使用 OrSpecification 将两者结合起来
        var combinedSpec = dotNetSpec.Or(architectureSpec);

        AddFilteringQuery(combinedSpec.FilterQuery!);

        AddOrderByDescendingQuery(post => post.Id);
    }
}
```

另一个例子，我们选择既是近期发布又具有高参与度的帖子：

```csharp
public class HighEngagementRecentPostSpecification : Specification<Post>
{
    public HighEngagementRecentPostSpecification(int daysBack = 7,
        int minLikes = 100, int minComments = 30)
    {
        var recentSpec = new RecentPostSpecification(daysBack);
        var highEngagementSpec = new HighEngagementPostSpecification(minLikes, minComments);

        // 使用 AndSpecification 将两者结合起来
        var combinedSpec = recentSpec.And(highEngagementSpec);

        AddFilteringQuery(combinedSpec.FilterQuery!);

        AddOrderByDescendingQuery(post => post.Likes.Count + post.Comments.Count);
    }
}
```

这使我们能够灵活地重用现有规约来形成新的规约。

注意：此处无需添加 **AddIncludeQuery(post => post.Category);** 等 Include 查询，因为我们没有在 Specification 本身中具体化数据库查询，而是使用了 Select Projection：

如果你使用仓储模式并希望有一个通用的方法来应用规约，你有两种选择：

- 添加 Include 查询，但这会导致 WhereAsync 方法从数据库返回过多数据
- 将带有映射函数的委托推送到仓储中的 WhereAsync 方法

这两种方法都有其优缺点；这就是为什么我更喜欢在不使用仓储模式的情况下使用 EF Core。

因此，我们的 API 端点（或应用程序处理程序或服务，取决于项目复杂性）将像这样简单：

```csharp
public class GetDotNetAndArchitecturePostsEndpoint : IEndpoint
{
    public void MapEndpoint(WebApplication app)
    {
        app.MapGet("/api/social-media/dotnet-architecture-posts", Handle);
    }

    private static async Task<IResult> Handle(
        ApplicationDbContext dbContext,
        CancellationToken cancellationToken)
    {
        var specification = new DotNetAndArchitecturePostSpecification();

        var response = await dbContext
            .ApplySpecification(specification)
            .Select(post => post.ToDto())
            .ToListAsync(cancellationToken);

        return Results.Ok(response);
    }
}
```

## 总结

规约模式是一个强大的工具，可用于在 .NET 项目中构建灵活且可重用的数据库查询。通过将筛选器、包含项和排序规则定义为 Specification 类，可以避免大型、难以维护的存储库所带来的问题。

使用 EF Core，你无需使用仓储模式——你可以将你的规约直接应用于 DbContext。

这种方法：

- 保持代码库整洁，易于修改
- 使查询可在应用程序的多个部分重复使用
- 允许你组合和复合规范以应对高级场景
- 帮助你独立于数据库测试查询逻辑

每当你发现自己向仓储中添加越来越多的方法或编写重复的查询逻辑时，请考虑使用规约模式。它将帮助你的项目以健康、可维护的方式成长。

今天就到这里。期待与你再次相见。
