# [我如何将 EF Core 查询从 30 秒优化到 30 毫秒](https://antondevtips.com/blog/ef-core-query-optimization-from-30-seconds-to-30-milliseconds)

性能对于任何应用程序都至关重要。

开发者经常在缓慢的数据库查询上添加缓存层。他们只是在掩盖症状，而不是解决根本问题。

今天，我们将接受一个挑战：如何优化一个缓慢的真实世界 EF Core 查询。EF Core 提供了出色的工具，但如果使用不当，可能会导致查询变慢。

我将向你展示我如何一步步将 EF Core 查询从无法接受的 **30 秒**优化到飞快的 **30 毫秒**。

让我们开始吧！

## 挑战与慢查询

我们将探索一个社交媒体平台，其中包含以下实体：

```csharp
public class User
{
    public int Id { get; set; }
    public string Username { get; set; } = null!;
    public ICollection<Post> Posts { get; set; } = new List<Post>();
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
}

public class Post
{
    public int Id { get; set; }
    public string Content { get; set; } = null!;
    public int UserId { get; set; }
    public User User { get; set; } = null!;
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    public ICollection<Like> Likes { get; set; } = new List<Like>();
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Comment
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } = null!;
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
    public string Text { get; set; } = null!;
    public DateTime CreatedAt { get; set; }
}

public class Like
{
    public int Id { get; set; }
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
}
```

这些实体具有以下关系：

- 用户：每个用户有很多文章和很多评论。
- 评论：每条评论都属于一个用户并链接到一篇文章。
- 分类：文章已分类。
- 文章：每篇文章都有一个分类，并且可以有多个点赞。
- 点赞：每个点赞都与一篇文章相关联。

这是我们的挑战要求：

- 选择过去 7 天内在 ".NET" 分类下的文章中发表评论最多的前 5 位用户。

对于每个用户返回：

- 用户 ID
- 用户名
- 用户评论数量（仅统计过去 7 天内对 ".NET" 文章的评论）
- 每个用户评论最多的点赞数排名前 3 的 ".NET" 文章（文章 ID，点赞数）

这是最初非常缓慢的查询：

```csharp
public List<ActiveUserDto> GetTopCommenters_Slow()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 1. 预加载每个用户及其所有评论 → 文章 → 分类 → 点赞
    var users = _dbContext.Users
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Category)
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Likes)
        .ToList();

    var result = new List<ActiveUserDto>();

    foreach (var u in users)
    {
        // 2. 过滤最近的 ".NET" 评论
        var comments = u.Comments
            .Where(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
            .ToList();

        var commentsCount = comments.Count;
        if (commentsCount == 0)
        {
            continue;
        }

        // 3. 按点赞数排名前3的文章
        var topPosts = comments
            .GroupBy(c => c.Post)
            .Select(g => new PostDto(
                g.Key.Id,
                _dbContext.Likes.Count(l => l.PostId == g.Key.Id)))
            .OrderByDescending(p => p.LikesCount)
            .Take(3)
            .ToList();

        // 4. 获取这3篇文章的最新2条评论
        var topPostIds = topPosts.Select(p => p.PostId).ToList();
        
        var recentTexts = _dbContext.Comments
            .Where(c =>
                c.UserId == u.Id &&
                c.CreatedAt >= since &&
                topPostIds.Contains(c.PostId))
            .OrderByDescending(c => c.CreatedAt)
            .Take(2)
            .Select(c => c.Text)
            .ToList();

        result.Add(new ActiveUserDto(
            u.Id, u.Username,
            commentsCount,
            topPosts,
            recentTexts));
    }

    // 5. 最终前5条数据
    return result
        .OrderByDescending(x => x.CommentsCount)
        .Take(5)
        .ToList();
}
```

这种实现方式会将所有数据急切地加载到内存中，然后完全在客户端执行大量的筛选、排序和聚合操作。这种方法导致大量数据集从数据库传输到应用程序，消耗大量内存并显著降低性能。

这个查询在 Postgres 数据库中运行需要 29-30 秒。

> 注意：每个基准测试结果都取决于你的 PC 硬件以及数据库提供程序和位置

## 优化 1：预筛选用户

我们加载了每个用户及其所有评论，包括那些在过去一周内未对".NET"文章发表评论的用户。

让我们添加一个筛选器，只返回那些确实在".NET"文章上发表过评论的用户。

这个简单的过滤器减少了从数据库检索的记录数量，从而使其他请求变得更快：

```csharp
/// <summary>
/// 优化 1：预筛选在过去一周内有任何 .NET 评论的用户
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization1_PreFilter()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 只过滤在过去一周内有任何 .NET 评论的用户
    var users = _dbContext.Users
        .Where(u => u.Comments
            .Any(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET"))
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Category)
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Likes)
        .ToList();

    // 与慢查询中的相同代码
}
```

让我们运行查询并比较性能：

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_1.png)

虽然我们只减少了 1 秒，但在更大的数据集上可能会有更好的表现。

让我们进一步改进。

## 优化 2：限制前 5 名用户

在上一步我们筛选出用户后，下一步是告诉数据库只按活动量返回前 5 位评论者。这样，我们永远不会加载超过 5 个我们真正关心的用户。

```csharp
/// <summary>
/// 优化 2：限制用户（按评论数倒序取前5名）
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization2_LimitUsers()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 计算评论数并取前5名
    var users = _dbContext.Users
        .Where(u => u.Comments
            .Any(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET"))
        .OrderByDescending(u => u.Comments
            .Count(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
        )
        .Take(5)
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Category)
        .Include(u => u.Comments)
            .ThenInclude(c => c.Post)
                .ThenInclude(p => p.Likes)
        .ToList();

    // 与慢查询中的相同代码
}
```

我们按评论数最多对用户进行排序，并只返回前 5 名用户。

运行基准测试后，我们发现时间从 27 秒减少到了 17 秒。

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_2.png)

这是一个显著的性能提升。

## 优化 3：限制 JOIN 的数量

与其将所有表都连接起来：users、comments、posts、categories、likes - 不如只连接所需的数据：

```csharp
/// <summary>
/// 优化 3：查询时过滤子评论
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization3_FilterComments()
{
    var since = DateTime.UtcNow.AddDays(-7);
 
    // 只包含匹配我们过滤条件的评论
    var users = _dbContext.Users
        .Include(u => u.Comments.Where(c =>
            c.CreatedAt >= since &&
            c.Post.Category.Name == ".NET"))
        .ThenInclude(c => c.Post)
            .ThenInclude(p => p.Likes)
        .Where(u => u.Comments.Any())
        .OrderByDescending(u => u.Comments
            .Count(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
        )
        .Take(5)
        .ToList();

    var result = new List<ActiveUserDto>();
    foreach (var u in users)
    {
        // u.Comments 现在只包含最近的 .NET 评论
        var comments = u.Comments.ToList();
        var commentsCount = comments.Count;

        var topPosts = comments
            .GroupBy(c => c.Post)
            .Select(g => new PostDto(
                g.Key.Id,
                g.Key.Likes.Count))
            .OrderByDescending(p => p.LikesCount)
            .Take(3)
            .ToList();

        var topPostIds = topPosts.Select(p => p.PostId).ToList();
        var recentTexts = _dbContext.Comments
            .Where(c =>
                c.UserId == u.Id &&
                c.CreatedAt >= since &&
                topPostIds.Contains(c.PostId))
            .OrderByDescending(c => c.CreatedAt)
            .Take(2)
            .Select(c => c.Text)
            .ToList();

        result.Add(new ActiveUserDto(
            u.Id,
            u.Username,
            commentsCount,
            topPosts,
            recentTexts));
    }

    return result;
}
```

运行基准测试后，你可以看到我们已经降至 10 秒，并且消耗的内存几乎减少了一半：

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_3.png)

## 优化 4：仅映射所需列

让我们进一步优化，只选择需要的列。

首先，我们只请求前 5 个用户及其总评论数。然后，在一个循环中，我们运行两个小查询：

- 他们评论最多的前 3 个文章（按点赞数）。
- 那些热门文章上的最新 2 条评论文本。

```csharp
/// <summary>
/// 优化 4： 仅映射所需列以此避免深层嵌套子查询。
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization4_Projection()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 获取用户及其评论计数
    var topUsers = _dbContext.Users
        .AsNoTracking() // 添加 AsNoTracking 以提高读取性能
        .Where(u => u.Comments.Any(c =>
            c.CreatedAt >= since &&
            c.Post.Category.Name == ".NET"))
        .Select(u => new
        {
            u.Id,
            u.Username,
            CommentsCount = u.Comments.Count(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
        })
        .OrderByDescending(u => u.CommentsCount)
        .Take(5)
        .ToList();

    var result = new List<ActiveUserDto>();

    foreach (var user in topUsers)
    {
        // 使用映射的方式获取最多赞的文章
        var topPosts = _dbContext.Comments
            .AsNoTracking()
            .Where(c =>
                c.UserId == user.Id &&
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
            .GroupBy(c => c.PostId)
            .Select(g => new
            {
                PostId = g.Key,
                LikesCount = _dbContext.Posts
                    .Where(p => p.Id == g.Key)
                    .Select(p => p.Likes.Count)
                    .FirstOrDefault()
            })
            .OrderByDescending(p => p.LikesCount)
            .Take(3)
            .Select(p => new PostDto(p.PostId, p.LikesCount))
            .ToList();

        // 获取文章 ID
        var topPostIds = topPosts.Select(p => p.PostId).ToList();

        // 获取该用户在热门文章上的最新评论
        var recentTexts = _dbContext.Comments
            .AsNoTracking()
            .Where(c =>
                c.UserId == user.Id &&
                c.CreatedAt >= since &&
                topPostIds.Contains(c.PostId))
            .OrderByDescending(c => c.CreatedAt)
            .Take(2)
            .Select(c => c.Text)
            .ToList();

        // 添加到结果列表
        result.Add(new ActiveUserDto(
            user.Id,
            user.Username,
            user.CommentsCount,
            topPosts,
            recentTexts
        ));
    }

    return result;
}
```

尽管存在 N+1 查询问题，我们已经从 10 秒优化到了 40 毫秒：

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_4.png)

我们在这里向数据库发送了 1 + (2 × 5) = 11 个简单调用。

现在是时候摆脱循环，尝试在单个查询中完成所有操作。

## 优化 5：单查询映射

让我们尝试将过滤、计数、分组、排序和分页都放入一个 LINQ 语句中：

```csharp
/// <summary>
/// 优化 5：单查询映射
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization5_OneQuery()
{
    var since = DateTime.UtcNow.AddDays(-7);

    var projected = _dbContext.Users
        // 1. 仅获取在过去 7 天内至少有 1 条 “.NET” 评论的用户
        .Where(u => u.Comments
            .Any(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET"))

        // 2. 映射字段
        .Select(u => new
        {
            u.Id,
            u.Username,
            CommentsCount = u.Comments
                .Count(c =>
                    c.CreatedAt >= since &&
                    c.Post.Category.Name == ".NET"),

            // 3. 按点赞数排序前 3 名的文章
            TopPosts = u.Comments
                .Where(c =>
                    c.CreatedAt >= since &&
                    c.Post.Category.Name == ".NET")
                .GroupBy(c => new { c.Post.Id, c.Post.Likes.Count })
                .Select(g => new { g.Key.Id, LikesCount = g.Key.Count })
                .OrderByDescending(p => p.LikesCount)
                .Take(3),

            // 4. 获取最新的 2 条评论
            RecentComments = u.Comments
                .Where(c =>
                    c.CreatedAt >= since &&
                    c.Post.Category.Name == ".NET" &&
                    // 子查询用于过滤满足条件的文章
                    u.Comments
                        .Where(d =>
                            d.CreatedAt >= since &&
                            d.Post.Category.Name == ".NET")
                        .GroupBy(d => d.PostId)
                        .OrderByDescending(g => g.Count())
                        .Take(3)
                        .Select(g => g.Key)
                        .Contains(c.PostId))
                .OrderByDescending(c => c.CreatedAt)
                .Take(2)
                .Select(c => c.Text)
        })

        // 5. 排序并获取前 5 条数据
        .OrderByDescending(x => x.CommentsCount)
        .Take(5)

        // 6. 转换到 DTO
        .Select(x => new ActiveUserDto(
            x.Id,
            x.Username,
            x.CommentsCount,
            x.TopPosts
                .Select(p => new PostDto(p.Id, p.LikesCount))
                .ToList(),
            x.RecentComments.ToList()
        ))
        .ToList();

    return projected;
}
```

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_5.png)

有趣的是，我们没有执行 11 个小查询，而是只执行了 1 个查询，却只获得了 3 毫秒的性能提升。

让我们检查自动生成的 SQL 查询：

```bash
SELECT u0.id, u0.username, u0.c, s0.id0, s0."Count", s1.text, s1.id, s1.id0, s1.id1
  FROM (
      SELECT u.id, u.username, (
          SELECT count(*)::int
          FROM devtips_optimization_challenge.comments AS c3
          INNER JOIN devtips_optimization_challenge.posts AS p1 ON c3.post_id = p1.id
          INNER JOIN devtips_optimization_challenge.categories AS c4 ON p1.category_id = c4.id
          WHERE u.id = c3.user_id AND c3.created_at >= @__since_0 AND c4.name = '.NET') AS c, (
          SELECT count(*)::int
          FROM devtips_optimization_challenge.comments AS c1
          INNER JOIN devtips_optimization_challenge.posts AS p0 ON c1.post_id = p0.id
          INNER JOIN devtips_optimization_challenge.categories AS c2 ON p0.category_id = c2.id
          WHERE u.id = c1.user_id AND c1.created_at >= @__since_0 AND c2.name = '.NET') AS c0
      FROM devtips_optimization_challenge.users AS u
      WHERE EXISTS (
          SELECT 1
          FROM devtips_optimization_challenge.comments AS c
          INNER JOIN devtips_optimization_challenge.posts AS p ON c.post_id = p.id
          INNER JOIN devtips_optimization_challenge.categories AS c0 ON p.category_id = c0.id
          WHERE u.id = c.user_id AND c.created_at >= @__since_0 AND c0.name = '.NET')
      ORDER BY (
          SELECT count(*)::int
          FROM devtips_optimization_challenge.comments AS c1
          INNER JOIN devtips_optimization_challenge.posts AS p0 ON c1.post_id = p0.id
          INNER JOIN devtips_optimization_challenge.categories AS c2 ON p0.category_id = c2.id
          WHERE u.id = c1.user_id AND c1.created_at >= @__since_0 AND c2.name = '.NET') DESC
      LIMIT @__p_1
  ) AS u0
  LEFT JOIN LATERAL (
      SELECT s.id0, s."Count"
      FROM (
          SELECT p2.id AS id0, (
              SELECT count(*)::int
              FROM devtips_optimization_challenge.likes AS l
              WHERE p2.id = l.post_id) AS "Count"
          FROM devtips_optimization_challenge.comments AS c5
          INNER JOIN devtips_optimization_challenge.posts AS p2 ON c5.post_id = p2.id
          INNER JOIN devtips_optimization_challenge.categories AS c6 ON p2.category_id = c6.id
          WHERE u0.id = c5.user_id AND c5.created_at >= @__since_0 AND c6.name = '.NET'
      ) AS s
      GROUP BY s.id0, s."Count"
      ORDER BY s."Count" DESC
      LIMIT 3
  ) AS s0 ON TRUE
  LEFT JOIN LATERAL (
      SELECT c7.text, c7.id, p3.id AS id0, c8.id AS id1, c7.created_at
      FROM devtips_optimization_challenge.comments AS c7
      INNER JOIN devtips_optimization_challenge.posts AS p3 ON c7.post_id = p3.id
      INNER JOIN devtips_optimization_challenge.categories AS c8 ON p3.category_id = c8.id
      WHERE u0.id = c7.user_id AND c7.created_at >= @__since_0 AND c8.name = '.NET' AND c7.post_id IN (
          SELECT c9.post_id
          FROM devtips_optimization_challenge.comments AS c9
          INNER JOIN devtips_optimization_challenge.posts AS p4 ON c9.post_id = p4.id
          INNER JOIN devtips_optimization_challenge.categories AS c10 ON p4.category_id = c10.id
          WHERE u0.id = c9.user_id AND c9.created_at >= @__since_0 AND c10.name = '.NET'
          GROUP BY c9.post_id
          ORDER BY count(*)::int DESC
          LIMIT 3
      )
      ORDER BY c7.created_at DESC
      LIMIT 2
  ) AS s1 ON TRUE
  ORDER BY u0.c0 DESC, u0.id, s0."Count" DESC, s0.id0, s1.created_at DESC, s1.id, s1.id0
```

这看起来很糟糕，并且容易受到笛卡尔爆炸的影响。让我们解决这个问题。

## 优化 6：拆分查询

当你在单个查询中映射深层对象图时，EF Core 通常会生成包含多个 JOIN 的庞大 SQL 语句。这可能导致"笛卡尔爆炸"，即相同的数据在多行中重复，从而增加传输大小并降低执行速度。

使用 **.AsSplitQuery()** 时，EF Core 会将一个大型查询分解为多个更简单的 SQL 语句。这会导致需要额外的请求来获取 JOIN 的数据：

```csharp
/// <summary>
/// 优化 6：拆分查询以避免笛卡尔爆炸
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization6_SplitQuery()
{
    var since = DateTime.UtcNow.AddDays(-7);

    var projected = _dbContext.Users
        // 优化 5 中相同的代码
        .AsSplitQuery()
        .ToList();

    return projected;
}
```

但是结果却令人惊讶：

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_6.png)

现在我们的查询运行时间为 51 毫秒，这甚至比我们使用 11 个 SQL 请求的方法还要慢。

让我们尝试一些不同的方法来进一步优化我们的查询。

## 优化 7：三阶段查询

让我们尝试分三个阶段来构建我们的查询：

- 第一阶段：获取评论数和热门文章的前 5 名用户
- 第二阶段：按点赞数获取这些用户的顶级文章
- 第三阶段：获取每个用户在其热门文章上的最新评论

```csharp
/// <summary>
/// 优化 7：三阶段查询
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization7_ThreePhaseOptimized()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 第一阶段：获取评论数和热门文章的前 5 名用户
    // 我们首先确定在过去 7 天内对 .NET 文章发表评论的用户
    // 并根据评论数获取前 5 名
    var topUsers = _dbContext.Users
        .Where(u => u.Comments.Any(c =>
            c.CreatedAt >= since &&
            c.Post.Category.Name == ".NET"))
        .Select(u => new
        {
            u.Id,
            u.Username,
            CommentsCount = u.Comments.Count(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET")
        })
        .OrderByDescending(x => x.CommentsCount)
        .Take(5)
        .ToList();

    // 获取用户 ID 以便在后续查询中使用
    var userIds = topUsers.Select(u => u.Id).ToArray();

    // 获取这些用户按点赞数排序的热门文章
    // 这避免了单查询方法中的笛卡尔爆炸
    var topPostsPerUser = _dbContext.Comments
        .Where(c =>
            userIds.Contains(c.UserId) &&
            c.CreatedAt >= since &&
            c.Post.Category.Name == ".NET")
        .GroupBy(c => new { c.UserId, c.PostId })
        .Select(g => new { g.Key.UserId, g.Key.PostId, LikesCount = g.First().Post.Likes.Count })
        .ToList()
        .GroupBy(x => x.UserId)
        .ToDictionary(
            g => g.Key,
            g => g.OrderByDescending(x => x.LikesCount)
                .Take(3)
                .Select(x => new PostDto(x.PostId, x.LikesCount))
                .ToList()
        );

    // 获取热门文章的 ID
    var allTopPostIds = topPostsPerUser
        .SelectMany(kvp => kvp.Value.Select(p => p.PostId))
        .Distinct()
        .ToArray();

    // 获取每个用户在其热门文章上的最新评论
    var recentCommentsPerUser = _dbContext.Comments
        .Where(c =>
            userIds.Contains(c.UserId) &&
            c.CreatedAt >= since &&
            allTopPostIds.Contains(c.PostId))
        .OrderByDescending(c => c.CreatedAt)
        .Select(c => new { c.UserId, c.Text, c.CreatedAt })
        .ToList()
        .GroupBy(c => c.UserId)
        .ToDictionary(
            g => g.Key,
            g => g.OrderByDescending(c => c.CreatedAt)
                .Take(2)
                .Select(c => c.Text)
                .ToList()
        );

    // 将查询结果合并
    var result = topUsers
        .Select(u => new ActiveUserDto(
            u.Id,
            u.Username,
            u.CommentsCount,
            topPostsPerUser.TryGetValue(u.Id, out var posts) ? posts : [],
            recentCommentsPerUser.TryGetValue(u.Id, out var comments) ? comments : []
        ))
        .ToList();

    return result;
}
```

让我们运行基准测试并查看结果：

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_7.png)

现在我们降到了 31 毫秒。

## 优化 8：两阶段查询

让我们尝试运行这两个查询：

- 第一阶段：获取前 5 名用户、他们的前 3 篇.NET 文章和评论数量
- 第二阶段：获取每个用户的最新 2 条评论

```csharp
/// <summary>
/// 优化 8：两阶段查询
/// </summary>
public List<ActiveUserDto> GetTopCommenters_Optimization8_TwoPhaseOptimized()
{
    var since = DateTime.UtcNow.AddDays(-7);

    // 第一阶段：获取前 5 名用户、他们的前 3 篇.NET 文章和评论数量
    var summaries = _dbContext.Users
        .Where(u => u.Comments
            .Any(c =>
                c.CreatedAt >= since &&
                c.Post.Category.Name == ".NET"))
        .Select(u => new
        {
            u.Id,
            u.Username,
            CommentsCount = u.Comments
                .Count(c =>
                    c.CreatedAt >= since &&
                    c.Post.Category.Name == ".NET"),
            TopPosts = u.Comments
                .Where(c =>
                    c.CreatedAt >= since &&
                    c.Post.Category.Name == ".NET")
                .GroupBy(c => c.PostId)
                .Select(g => new
                {
                    PostId = g.Key,
                    LikesCount = _dbContext.Likes
                        .Count(l => l.PostId == g.Key)
                })
                .OrderByDescending(p => p.LikesCount)
                .Take(3)
                .ToList()
        })
        .OrderByDescending(x => x.CommentsCount)
        .Take(5)
        .ToList();

    var userIds = summaries.Select(x => x.Id).ToArray();

    var postIds = summaries
        .SelectMany(s => s.TopPosts.Select(tp => tp.PostId))
        .Distinct()
        .ToArray();

    // 第二阶段：获取每个用户的最新 2 条评论
    var recentCommentsLookup = _dbContext.Comments
        .Where(c =>
            c.CreatedAt >= since &&
            c.Post.Category.Name == ".NET" &&
            userIds.Contains(c.UserId) &&
            postIds.Contains(c.PostId))
        .GroupBy(c => new { c.UserId, c.PostId })
        .Select(g => new
        {
            g.Key.UserId,
            g.Key.PostId,
            LatestTwo = g
                .OrderByDescending(c => c.CreatedAt)
                .Take(2)
                .Select(c => c.Text)
                .ToList()
        })
        .ToList()
        .ToLookup(x => x.UserId, x => x);

    // 将查询结果合并
    var result = summaries
        .Select(s => new ActiveUserDto(
            s.Id,
            s.Username,
            s.CommentsCount,
            s.TopPosts
                .Select(tp => new PostDto(tp.PostId, tp.LikesCount))
                .ToList(),
            recentCommentsLookup[s.Id]
                .SelectMany(x => x.LatestTwo)
                .OrderByDescending(text => text)
                .Take(2)
                .ToList()
        ))
        .ToList();

    return result;
}
```

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_8.png)

现在我们降至约 29-30 毫秒，这是迄今为止最快的速度。

## 横向比较所有基准测试结果

![基准测试结果](https://antondevtips.com/media/code_screenshots/efcore/optimization_challenge/img_8.png)

优化 8 是最快的，约 30 毫秒，但使用了约 228 KB。

优化方案 5（"单次查询"）运行时间约为 38 毫秒（慢 9 毫秒），但仅分配约 104 KB 内存，不到优化方案 8 内存使用量的一半。

实际上，我会从优化方案 5 开始，因为它是最平衡的：速度非常快且分配较低。

随着数据集的增长，始终监控执行时间和内存使用情况，如果发现单查询计划性能下降，请准备切换到多阶段方法（优化 7/8）。

## 总结

今天我们进行了 EF Core 挑战，优化一个非常缓慢的数据库查询。

我们从 30 秒开始，最终采用了一个两阶段模式，运行时间约为 30 毫秒。

以下是关键要点：

- 筛选数据（优化 1、2、3），这样你只加载需要的数据。
- 两阶段映射（优化 4、7、8）将大型工作分解为小型、专注的查询。
- 单查询映射（优化 5）在简单性、速度和低内存使用之间取得了平衡。
- 拆分查询（优化 6）在映射深度图时可以防止笛卡尔爆炸。

另外需要注意的是：考虑在查询中经常访问的字段上添加数据库索引——以使其更快。

最后，请记住黄金法则：先测量，再优化。通常，你会发现你的查询已经相当快，根本不需要优化。

使用日志或分析工具来识别你环境中的真正瓶颈，应用你在此处学到的有针对性的改进措施，并验证每个更改是否带来了预期的性能提升。

今天就到这里。期待与你再次相见。
