# [EF Core 中的预先加载](https://www.nikolatech.net/blogs/ef-core-eager-loading)

EF Core 提供了许多选项来适应你的用例，关键是充分利用其全部潜力。

其功能之一是通过不同的加载策略管理相关数据。

EF Core 支持 3 种加载相关实体的方式：

- **预加载** - 作为初始查询的一部分获取相关数据。
- **显式加载** - 仅在需要时手动加载相关数据。
- **延迟加载** - 推迟加载相关数据，直到实际访问时才加载。

在今天的文章中，我们将要了解一下预加载。

## 预加载

预加载是一种策略，EF Core 通过单个查询同时获取主实体和相关实体。

这有助于确保所有必需的数据立即可用，从而避免 N+1 查询问题并优化为更少的数据库往返次数。

另一方面，当你处理复杂对象时，如果获取大量数据却只是偶尔使用它，这种方式就不太理想，在这种情况下，其他策略可能更为适合。

在这个例子中，我创建了几个简单的实体：

```csharp
public class Blog
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public List<Tag> Tags { get; set; } = [];
    public List<Post> Posts { get; set; } = [];
}

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; }
    public Blog Blog { get; set; }
    public int BlogId { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public Blog Blog { get; set; }
    public int BlogId { get; set; }
    public List<Comment> Comments { get; set; }
}

public class Comment
{
    public int Id { get; set; }
    public string Content { get; set; }
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
}
```

Blog 实体包含 Posts 和 Tags，每个 Post 可以有多条 Comments。

## Include

你可以使用 **Include** 方法来指定要包含在查询结果中的相关数据：

```csharp
app.MapGet("include/", async (ApplicationDbContext dbContext) =>
{
    var blogs = await dbContext.Blogs
        .Include(b => b.Posts)
        .ToListAsync();

    return Results.Ok(blogs);
});
```

返回的博客将填充其 Posts 属性及相关文章：

```json
[
  {
    "id": 1,
    "createdAt": "2025-02-20T02:47:01.328624Z",
    "tags": [],
    "posts": [
      {
        "id": 1,
        "titles": "Porro nam ipsam itaque consequatur.",
        "blog": null,
        "blogId": 1,
        "comments": null
      },
      {
        "id": 2,
        "titles": "Id accusamus cupiditate eveniet fuga ullam consequatur provident molestiae.",
        "blog": null,
        "blogId": 1,
        "comments": null
      }
    ]
  },
  {
    "id": 2,
    "createdAt": "2024-08-23T03:24:11.221158Z",
    "tags": [],
    "posts": [
      {
        "id": 4,
        "title": "Sit maxime eaque est earum ut possimus distinctio deleniti.",
        "blog": null,
        "blogId": 2,
        "comments": null
      }
    ]
  }
]
```

## 多个 Include

你可以在单个查询中包含来自多个关系的相关数据：

```csharp
app.MapGet("multi-include/", async (ApplicationDbContext dbContext) =>
{
    var blogs = await dbContext.Blogs
        .Include(b => b.Tags)
        .Include(b => b.Posts)
        .ToListAsync();

    return Results.Ok(blogs);
});
```

在单个查询中预先加载多个集合导航可能会导致性能问题，也就是所谓的笛卡尔爆炸。

要了解更多信息，请查看我昨天的文章，我们在其中讨论了这个问题：Split Queries

## 筛选 Include

使用 **Include** 时，你还可以应用可枚举操作来筛选和排序结果：

```csharp
using (var context = new BloggingContext())
{
    var filteredBlogs = await context.Blogs
        .Include(
            blog => blog.Posts
                .Where(post => post.BlogId == 1)
                .OrderByDescending(post => post.Title)
                .Take(5))
        .ToListAsync();
}
```

每个包含的导航属性只允许一组唯一的筛选操作。

支持的操作有：

- Where
- OrderBy
- OrderByDescending
- ThenBy
- ThenByDescending
- Skip
- Take

注意：使用跟踪查询时，由于导航修复，筛选包含的结果可能与预期不符。

基本上，已被跟踪的实体即使不匹配筛选条件也可能出现，要防止这种情况，请使用 **AsNotTracking()** 或新的 **DbContext** 来避免。

有关更多信息，请查看官方文档：Filtered Include

## ThenInclude

你可以通过链式调用使用 **ThenInclude** 来加载多个级别的相关数据。

它允许你在单个查询中包含来自不同关系深度的数据。

这并不意味着你会得到冗余的连接，在大多数情况下，EF 在生成 SQL 时会合并这些连接。

## AutoInclude

EF Core 还支持使用 **[AutoInclude]** 特性或模型配置进行自动预加载：

```csharp
modelBuilder.Entity<Blog>()
    .Navigation(b => b.Posts)
    .AutoInclude();
```

这确保相关数据始终被加载，而无需在每个查询中指定 Include()。

如果在特定查询中，你不想通过在模型级别配置为自动包含的导航属性加载相关数据，可以在查询中使用 **IgnoreAutoIncludes** 方法。

```csharp
var blogs = await context.Blogs
    .IgnoreAutoIncludes()
    .ToListAsync();
```

使用此方法将停止加载所有配置为自动包含的导航属性。

注意：导航到拥有类型的属性也会按照惯例配置为自动包含，并且使用 **IgnoreAutoIncludes** 不会阻止它们被包含。

## 总结

EF Core 提供了灵活的策略来高效加载相关数据，其中预加载是一个强大的选项，可以在一个查询中获取所有需要的实体，从而避免多次数据库往返。

虽然预先加载简化了数据检索，但需要注意潜在的性能陷阱，如笛卡尔爆炸或使用筛选包含时的跟踪怪癖。

**ThenInclude** 和 **AutoInclude** 等功能为复杂关系加载提供了进一步的控制，帮助你定制查询以适应应用程序的特定需求。

今天就到这里。期待与你再次相见。
