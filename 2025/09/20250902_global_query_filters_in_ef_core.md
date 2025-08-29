# [EF Core 中的全局查询过滤器](https://antondevtips.com/blog/global-query-filters-in-ef-core)

## 什么是全局查询过滤器？

Entity Framework Core（EF Core）中的全局查询过滤器是一个强大功能，可用于有效管理数据访问模式。

全局查询过滤器是在 EF Core 实体模型上应用的 LINQ 查询谓词。这些过滤器会自动应用于涉及相应实体的所有查询。这在多租户应用或需要软删除的场景中特别有用。

## 用于软删除用例的查询过滤器

让我们来探讨一个全局查询过滤器特别有用的用例——实体软删除。在某些应用中，实体不能从数据库中完全删除，例如为了统计和历史记录的目的，或者为了确保相关数据保持不变，例如：被外键引用。针对这种用例的解决方案就是软删除。

软删除通过为需要的实体在数据库表中添加一个 **is_deleted** 列来实现。每当一个实体被视为已删除时——该列被设置为 true 。在大多数应用的数据库查询中，“已删除”的实体在读取操作中应被忽略，不应对最终用户可见。

让我们看看以下实体的定义：

```csharp
public class Author
{
    public required Guid Id { get; set; }
    public required string Name { get; set; }
    public required string Country { get; set; }
    public required List<Book> Books { get; set; } = [];
}

public class Book
{
    public required Guid Id { get; set; }
    public required string Title { get; set; }
    public required int Year { get; set; }
    public required bool IsDeleted { get; set; }
    public required Guid TenantId { get; set; }
    public required Author Author { get; set; }
}
```

我们需要创建并设置我们的 DbContext，并添加一个全局查询过滤器：

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Author> Authors { get; set; } = default!;
    
    public DbSet<Book> Books { get; set; } = default!;
    
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
            .HasQueryFilter(x => !x.IsDeleted);
        
        base.OnModelCreating(modelBuilder);
        
        modelBuilder.Entity<Author>(entity =>
        {
            entity.ToTable("authors");

            entity.HasKey(x => x.Id);
            entity.HasIndex(x => x.Name);
            
            entity.Property(x => x.Id).IsRequired();
            entity.Property(x => x.Name).IsRequired();
            entity.Property(x => x.Country).IsRequired();
            
            entity.HasMany(x => x.Books)
                .WithOne(x => x.Author);
        });
        
        modelBuilder.Entity<Book>(entity =>
        {
            entity.ToTable("books");

            entity.HasKey(x => x.Id);
            entity.HasIndex(x => x.Title);
            
            entity.Property(x => x.Id).IsRequired();
            entity.Property(x => x.Title).IsRequired();
            entity.Property(x => x.Year).IsRequired();
            entity.Property(x => x.IsDeleted).IsRequired();

            entity.HasOne(x => x.Author)
                .WithMany(x => x.Books);
        });
    }
}
```

在这段代码中，我们对 **Book** 实体的 **IsDeleted** 属性应用了查询筛选器：

```csharp
modelBuilder.Entity<Book>()
    .HasQueryFilter(x => !x.IsDeleted);
```

在这里我们从结果查询中过滤掉了所有被软删除的书籍。当从 DbContext 查询书籍时，这个查询过滤器会被自动应用。让我们来看下面的 API 接口：

```csharp
app.MapGet("/api/books", async (ApplicationDbContext dbContext) =>
{
    var nonDeletedBooks = await dbContext.Books.ToListAsync();
    return Results.Ok(nonDeletedBooks);
});
```

每次查询图书时，我们只获取那些未被删除的，因此在所有 DbContext 查询中都不需要使用 LINQ 的 Where 语句。

但是在某些情况下，我们可能需要访问所有实体并忽略查询筛选。EF Core 为这种情况提供了一个名为 **IgnoreQueryFilters** 的特殊方法：

```csharp
app.MapGet("/api/all-books", async (ApplicationDbContext dbContext) =>
{
    var allBooks = await dbContext.Books
        .IgnoreQueryFilters()
        .Where(x => x.IsDeleted)
        .ToListAsync();
    
    return Results.Ok(allBooks);
});
```

这样就会从数据库中检索出所有书籍，并且完全忽略对 Book 实体的查询过滤器。

## 多租户应用的查询过滤器

全局查询过滤器的另一个有用场景是多租户。多租户应用是为不同客户共享同一套软件的应用。为客户存储的所有数据都不应对其他客户可见。

让我们通过将所有数据存储在相同的数据库和表中，来探索多租户的最简单实现。

首先，我们需要在 **Books** 实体上添加一个 **TenantId** 属性：

```csharp
public class Book
{
    // 此处省略其他属性
    
    public required Guid TenantId { get; set; }
}
```

其次，我们需要更新 DbContext：

```csharp
public class ApplicationDbContext : DbContext
{
    private readonly Guid? _currentTenantId;
    
    public DbSet<Author> Authors { get; set; } = default!;
    
    public DbSet<Book> Books { get; set; } = default!;
    
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ITenantService tenantService) : base(options)
    {
        _currentTenantId = tenantService.GetCurrentTenantId();
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
            .HasQueryFilter(x => !x.IsDeleted && x.TenantId == _currentTenantId);
        
        base.OnModelCreating(modelBuilder);
        
        // 剩余的代码保持不变
    }
}
```

在这段代码中，我们更新了查询筛选器并添加了一个 **x.TenantId == _currentTenantId** 语句。由于我们为每个请求创建 DbContext——可以从请求中注入当前租户 ID（访问我们应用中数据的客户的标识符）。

下面是一个简单的租户服务实现示例，它从 HTTP 请求头中检索租户 ID：

```csharp
public interface ITenantService
{
    Guid? GetCurrentTenantId();
}

public class TenantService : ITenantService
{
    private readonly Guid? _currentTenantId;
    
    public TenantService(IHttpContextAccessor accessor)
    {
        var headers = accessor.HttpContext?.Request.Headers;

        _currentTenantId = headers.TryGetValue("Tenant-Id", out var value) is true
            ? Guid.Parse(value.ToString())
            : null;
    }

    public Guid? GetCurrentTenantId() => _currentTenantId;
}
```

现在让我们创建一个相应的 API 接口：

```csharp
app.MapGet("/api/tenant-books", async (ApplicationDbContext dbContext) =>
{
    var tenantBooks = await dbContext.Books.ToListAsync();
    return Results.Ok(tenantBooks);
});
```

在每次读取查询时都会应用租户 ID 全局查询过滤器以确保存储数据的完整性。因此，在调用此端点时，每个客户只能检索到他们自己的书籍。

## 总结

EF Core 中的全局查询过滤器是一项强大的功能，可在整个应用程序中一致地强制执行数据访问规则。它们在多租户架构和软删除等场景中特别有用，能够确保在所有读取操作中自动应用过滤查询。通过将这些过滤器应用于 EF Core 实体模型，可以显著简化数据访问代码，确保数据完整性，并降低在读取操作中忘记应用重要过滤器的风险。

今天就到这里。期待与你再次相见。
