# [别再把 JSON 存为文本了：EF 10 让它成为原生类型](https://medium.com/c-sharp-programming/ef-core-10-sql-server-native-json-e72a031e6f31)

SQL Server 2025 和 Azure SQL 将引入原生 json 类型，而 EF Core 10 知道如何使用它。不再需要将 JSON 塞进 nvarchar(max) 。不再需要祈祷数据有效。

## 为什么这么做？

答案是速度。

JSON 存储在真正的数据类型中，因此读写速度更快。你可以对它进行索引、过滤和部分更新，而无需处理文本。

- 安全性：数据库保证 JSON 的有效性。错误的有效载荷不会悄悄混入。
- 更更简洁的模型：EF 10 理解数组和复杂类型。它将它们映射到 json 列而不是字符串。你的代码看起来像 C#，而不是字符串操作。

## EF Core 10 中的工作原理

目标 SQL Server 兼容级别 170+ 或使用 UseAzureSql() 。EF 将默认将数组和复杂类型映射到新的 json 列类型。

### 模型

```csharp
public class Blog
{
    public int Id { get; set; }
    public string[] Tags { get; set; } = [];
    public required BlogDetails Details { get; set; }
}

public class BlogDetails
{
    public string? Description { get; set; }
    public int Viewers { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .ComplexProperty(b => b.Details, b => b.ToJson());
}
```

### 结果表

```sql
CREATE TABLE [Blogs] (
    [Id] int NOT NULL IDENTITY,
    [Tags] json NOT NULL,
    [Details] json NOT NULL,
    CONSTRAINT [PK_Blogs] PRIMARY KEY ([Id])
);
```

### 像普通 C# 一样查询

LINQ 直接可用：

```csharp
var popularBlogs = await context.Blogs
    .Where(b => b.Details.Viewers > 3)
    .ToListAsync();
```

EF 会使用 JSON_VALUE(... RETURNING int) 将其转换为高效的 SQL。你既保持了强类型，又获得了真正的性能。

### 迁移说明

- 当前有使用 nvarchar(max) 列存储 JSON？迁移可以自动将它们转换为 json 。
- 想要继续使用字符串存储？请明确设置列类型或降低兼容性级别。
- 类型转换很重要。 json 与字符串或二进制之间的转换需要显式类型转换。没有免费的隐式魔法。

## Azure SQL 状态

目前，在 RC1 版本中，由于上游存在差距， json 类型在 Azure SQL 上暂时被禁用。

EF 团队计划在 EF 10.0 版本中启用此功能。如果你要部署到 Azure，请查看发行说明。

## 何时使用它

- 形状不断变化的日志和审计
- 多值属性，如标签、设置和功能标志
- 尚不需要自己表格的嵌套详细信息

## 资深开发者提示

升级后，请查看你的实体和最常用的查询。原生 JSON 可以改变你处理数据建模的方式。

考虑简化表格、添加 JSON 索引，并将更多过滤操作推送到数据库。从你最慢的查询开始，看看 JSON 索引是否有帮助。

## 代码示例

将其复制到一个空白项目中。你将获得 SQL Server 上的 JSON 类型的 EF Core 10、一个迁移、种子数据和一个微小的性能测试。

### 先决条件

- .NET 10 SDK
- SQL Server 2025 或 Azure SQL

### 创建项目

```bash
dotnet new console -n EfJsonQuickstart && cd EfJsonQuickstart
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 10.0.0-rc.1*
dotnet add package Microsoft.EntityFrameworkCore.Design  --version 10.0.0-rc.1*
```

创建 appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EfJsonQuickstart;Trusted_Connection=True;"
  }
}
```

如果这是一个全新的数据库，需要设置兼容级别为 170+：

```sql
ALTER DATABASE [EfJsonQuickstart] SET COMPATIBILITY_LEVEL = 170;
```

### 放入代码

将 Program.cs 替换为：

```csharp
using System.Diagnostics;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false)
    .AddEnvironmentVariables()
    .Build();

var cs = config.GetConnectionString("Sql")!;

var opt = new DbContextOptionsBuilder<AppDb>()
    .UseSqlServer(cs, o => o.EnableRetryOnFailure())
    .Options;

await using var db = new AppDb(opt);

// Apply migrations on startup (dev/demo only)
await db.Database.MigrateAsync();

// Seed a few rows if empty
if (!await db.Blogs.AnyAsync())
{
    var seed = Enumerable.Range(1, 2_000).Select(i => new Blog
    {
        Tags = new[] { "dotnet", "ef", i % 2 == 0 ? "json" : "sql" },
        Details = new BlogDetails { Description = $"Post #{i}", Viewers = i % 10 }
    });
    var swSeed = Stopwatch.StartNew();
    await db.Blogs.AddRangeAsync(seed);
    await db.SaveChangesAsync();
    swSeed.Stop();
    Console.WriteLine($"Seeded 2,000 rows in {swSeed.ElapsedMilliseconds} ms");
}

// Tiny perf check: JSON filter vs in-memory post-filter
var swDb = Stopwatch.StartNew();
var hot = await db.Blogs
    .Where(b => b.Details.Viewers > 5) // translated to JSON_VALUE(... RETURNING int)
    .Select(b => new { b.Id, b.Details.Viewers })
    .ToListAsync();
swDb.Stop();

var swMem = Stopwatch.StartNew();
var hotMem = (await db.Blogs.AsNoTracking().ToListAsync())
    .Where(b => b.Details.Viewers > 5)
    .Select(b => new { b.Id, b.Details.Viewers })
    .ToList();
swMem.Stop();

Console.WriteLine($"DB filter: {hot.Count} rows in {swDb.ElapsedMilliseconds} ms");
Console.WriteLine($"App filter: {hotMem.Count} rows in {swMem.ElapsedMilliseconds} ms");

public class AppDb : DbContext
{
    public AppDb(DbContextOptions<AppDb> options) : base(options) { }
    public DbSet<Blog> Blogs => Set<Blog>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // 映射数组 + 复杂类型到原生 json
        b.Entity<Blog>(e =>
        {
            e.Property(x => x.Tags)
             .HasColumnType("json")
             .IsRequired();

            e.ComplexProperty(x => x.Details, d =>
            {
                d.ToJson();
                d.IsRequired();
            });
        });

        // 可选：在你经常查询的 JSON 路径上创建索引
        // 使用迁移添加此索引：
        // b.Entity<Blog>().HasIndex("Details").HasDatabaseName("IX_Blogs_Details_Viewers");
    }
}

public class Blog
{
    public int Id { get; set; }
    public string[] Tags { get; set; } = [];
    public required BlogDetails Details { get; set; }
}

public class BlogDetails
{
    public string? Description { get; set; }
    public int Viewers { get; set; }
}
```

### 创建并运行迁移

```bash
dotnet tool update --global dotnet-ef
dotnet ef migrations add Init
dotnet ef database update
dotnet run
```

你应该会看到 EF 生成如下表：

```sql
CREATE TABLE [Blogs] (
  [Id] int NOT NULL IDENTITY,
  [Tags] json NOT NULL,
  [Details] json NOT NULL,
  CONSTRAINT [PK_Blogs] PRIMARY KEY ([Id])
);
```

以及类似如下的控制台输出：

```bash
Seeded 2000 rows in 450 ms
DB filter: 800 rows in 20 ms
App filter: 800 rows in 60 ms
```

### 可选：为热点字段添加 JSON 路径索引

添加一个带有目标索引的迁移（名称可能会随着最终 SQL Server 文档而改变）：

```csharp
migrationBuilder.Sql("""
CREATE INDEX IX_Blogs_Details_Viewers
ON dbo.Blogs (JSON_VALUE([Details], '$.Viewers' RETURNING int));
""");
```

今天就到这里。希望这些内容对你有帮助。
