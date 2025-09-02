# [如何在 ASP.NET Core 中使用 EF Core 实现审计追踪](https://antondevtips.com/blog/how-to-implement-audit-trail-in-asp-net-core-with-ef-core)

在现代 Web 应用程序中，出于监控、合规性和调试目的，可能需要跟踪数据的更改。这个过程被称为创建审计追踪，它允许开发人员查看谁进行了更改、何时进行了更改以及更改了什么。审计追踪提供了数据更改的历史记录。

今天，我们将探讨如何使用 Entity Framework Core (EF Core) 在 ASP.NET Core 应用程序中实现审计追踪。

## 我们将审计的应用程序

今天我们将为“图书”应用程序实现审计追踪，该应用程序包含以下实体：

- Books 书籍
- Authors 作者
- Users 用户

我发现，在所有需要审计的实体中包含以下属性会很有用：

```csharp
public interface IAuditableEntity
{
    DateTime CreatedAtUtc { get; set; }
    
    DateTime? UpdatedAtUtc { get; set; }
    
    string CreatedBy { get; set; }

    string? UpdatedBy { get; set; }
}
```

我们需要让所有可审计实体都继承此接口，例如 User 和 Book：

```csharp
public class User : IAuditableEntity
{
    public Guid Id { get; set; }
    public required string Email { get; set; }
    
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public string CreatedBy { get; set; } = null!;
    public string? UpdatedBy { get; set; }
}

public class Book : IAuditableEntity
{
    public required Guid Id { get; set; }
    public required string Title { get; set; }
    public required int Year { get; set; }
    public Guid AuthorId { get; set; }
    public Author Author { get; set; } = null!;
    
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public string CreatedBy { get; set; } = null!;
    public string? UpdatedBy { get; set; }
}
```

现在我们有几个选择，我们可以为每个实体手动实现审计追踪，或者采用一种自动应用于所有实体的实现方式。而我将向你展示第二种选择，因为它更健壮且更易于维护。

## 在 EF Core 中配置审计追踪实体

实现审计追踪的第一步是创建一个实体，用于将审计日志存储在单独的数据库表中。该实体应捕获详细信息，例如实体类型、主键、更改属性列表、旧值、新值以及更改的时间戳。

```csharp
public class AuditTrail
{
    public required Guid Id { get; set; }

    public Guid? UserId { get; set; }

    public User? User { get; set; }

    public TrailType TrailType { get; set; }

    public DateTime DateUtc { get; set; }

    public required string EntityName { get; set; }

    public string? PrimaryKey { get; set; }

    public Dictionary<string, object?> OldValues { get; set; } = [];

    public Dictionary<string, object?> NewValues { get; set; } = [];

    public List<string> ChangedColumns { get; set; } = [];
}
```

这里我们引用了一个 User 实体。根据你的应用程序需求，你可能需要此引用，也可能不需要。

每种审计追踪都可以是以下类型：

- 创建 (Create)
- 更新 (Update)
- 删除 (Delete)

```csharp
public enum TrailType : byte
{
    None = 0,
    Create = 1,
    Update = 2,
    Delete = 3
}
```

让我们看看如何在 EF Core 中配置审计追踪实体：

```csharp
public class AuditTrailConfiguration : IEntityTypeConfiguration<AuditTrail>
{
    public void Configure(EntityTypeBuilder<AuditTrail> builder)
    {
        builder.ToTable("audit_trails");
        builder.HasKey(e => e.Id);

        builder.HasIndex(e => e.EntityName);

        builder.Property(e => e.Id);

        builder.Property(e => e.UserId);
        builder.Property(e => e.EntityName).HasMaxLength(100).IsRequired();
        builder.Property(e => e.DateUtc).IsRequired();
        builder.Property(e => e.PrimaryKey).HasMaxLength(100);

        builder.Property(e => e.TrailType).HasConversion<string>();

        builder.Property(e => e.ChangedColumns).HasColumnType("jsonb");
        builder.Property(e => e.OldValues).HasColumnType("jsonb");
        builder.Property(e => e.NewValues).HasColumnType("jsonb");

        builder.HasOne(e => e.User)
            .WithMany()
            .HasForeignKey(e => e.UserId)
            .IsRequired(false)
            .OnDelete(DeleteBehavior.SetNull);
    }
}
```

我喜欢使用 JSON 列来表达 ChangedColumns 、 OldValues 和 NewValues 。在我的代码示例中，我使用了 Postgres 数据库。

如果你正在使用 SQLite 或其他不支持 json 列的数据库，你可以在你的实体中使用字符串类型，并创建一个 EF Core 转换，将对象序列化为字符串以保存到数据库中。当从数据库检索数据时，此转换会将 JSON 字符串反序列化为相应的.NET 类型。

在 Postgres 数据库中，当使用.NET 8 和 EF 8 时，你需要 **EnableDynamicJson** 才能在“jsonb”列中拥有动态 json ：

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);
dataSourceBuilder.EnableDynamicJson();

builder.Services.AddDbContext<ApplicationDbContext>((provider, options) =>
{
    var interceptor = provider.GetRequiredService<AuditableInterceptor>();

    options.EnableSensitiveDataLogging()
        .UseNpgsql(dataSourceBuilder.Build(), npgsqlOptions =>
        {
            npgsqlOptions.MigrationsHistoryTable("__MyMigrationsHistory", "devtips_audit_trails");
        })
        .AddInterceptors(interceptor)
        .UseSnakeCaseNamingConvention();
});
```

## 为所有可审计实体实现审计追踪

我们可以在 EF Core DbContext 中实现审计，该审计将自动应用于所有继承自 IAuditableEntity 的实体。但首先我们需要获取对这些实体执行创建、更新或删除操作的用户。

让我们定义一个 **CurrentSessionProvider** ，它将从当前 HttpRequest 的 ClaimsPrinciple 中检索当前用户标识符：

```csharp
public interface ICurrentSessionProvider
{
    Guid? GetUserId();
}

public class CurrentSessionProvider : ICurrentSessionProvider
{
    private readonly Guid? _currentUserId;

    public CurrentSessionProvider(IHttpContextAccessor accessor)
    {
        var userId = accessor.HttpContext?.User.FindFirstValue("userid");
        if (userId is null)
        {
            return;
        }

        _currentUserId = Guid.TryParse(userId, out var guid) ? guid : null;
    }

    public Guid? GetUserId() => _currentUserId;
}
```

你需要在 DI 中注册提供程序和 **IHttpContextAccessor** ：

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentSessionProvider, CurrentSessionProvider>();
```

要创建审计追踪，我们可以使用 EF Core 变更追踪器功能来获取已创建、更新或删除的实体。

我们需要将 **ICurrentSessionProvider** 注入到 DbContext 中，并重写 SaveChangesAsync 方法来创建审计追踪。

```csharp
public class ApplicationDbContext(
    DbContextOptions<ApplicationDbContext> options,
    ICurrentSessionProvider currentSessionProvider)
    : DbContext(options)
{
    public ICurrentSessionProvider CurrentSessionProvider => currentSessionProvider;
    
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new())
    {
        var userId = CurrentSessionProvider.GetUserId();
        
        SetAuditableProperties(userId);

        var auditEntries = HandleAuditingBeforeSaveChanges(userId).ToList();
        if (auditEntries.Count > 0)
        {
            await AuditTrails.AddRangeAsync(auditEntries, cancellationToken);
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

请注意，我们在调用 base.SaveChangesAsync 之前创建 AuditTrails ，以确保所有更改都在单个事务中持久化到数据库。

在上面的代码中，我们执行了两个操作：

- 设置可审计属性以创建、更新或删除记录
- 创建审计追踪记录

对于所有继承自 IAuditableEntity 的实体，我们设置 Created 和 Updated 字段。在某些情况下，更改可能不是由用户触发的，而是由代码触发的。在这种情况下，我们设置由“系统”执行了更改。

例如，这可以是一个后台作业、数据库数据初始化作业等。

```csharp
private void SetAuditableProperties(Guid? userId)
{
    const string systemSource = "system";
    foreach (var entry in ChangeTracker.Entries<IAuditableEntity>())
    {
        switch (entry.State)
        {
            case EntityState.Added:
                entry.Entity.CreatedAtUtc = DateTime.UtcNow;
                entry.Entity.CreatedBy = userId?.ToString() ?? systemSource;
                break;

            case EntityState.Modified:
                entry.Entity.UpdatedAtUtc = DateTime.UtcNow;
                entry.Entity.UpdatedBy = userId?.ToString() ?? systemSource;
                break;
        }
    }
}
```

现在我们来看看如何创建审计追踪记录。同样，我们遍历 IAuditableEntity 个实体，并选择那些已创建、已更新或已删除的实体：

```csharp
private List<AuditTrail> HandleAuditingBeforeSaveChanges(Guid? userId)
{
    var auditableEntries = ChangeTracker.Entries<IAuditableEntity>()
        .Where(x => x.State is EntityState.Added or EntityState.Deleted or EntityState.Modified)
        .Select(x => CreateTrailEntry(userId, x))
        .ToList();

    return auditableEntries;
}

private static AuditTrail CreateTrailEntry(Guid? userId, EntityEntry<IAuditableEntity> entry)
{
    var trailEntry = new AuditTrail
    {
        Id = Guid.NewGuid(),
        EntityName = entry.Entity.GetType().Name,
        UserId = userId,
        DateUtc = DateTime.UtcNow
    };

    SetAuditTrailPropertyValues(entry, trailEntry);
    SetAuditTrailNavigationValues(entry, trailEntry);
    SetAuditTrailReferenceValues(entry, trailEntry);

    return trailEntry;
}
```

审计追踪记录可以包含以下类型的属性：

- 普通属性（如图书的标题或出版年份）
- 引用属性（如 Book 的 Author）
- 导航属性（如作者的图书）

让我们看看如何向审计追踪添加**普通**属性：

```csharp
private static void SetAuditTrailPropertyValues(EntityEntry entry, AuditTrail trailEntry)
{
    // 忽略临时字段（例如：插入实体时将由 ef core 引擎自动分配的字段）
    foreach (var property in entry.Properties.Where(x => !x.IsTemporary))
    {
        if (property.Metadata.IsPrimaryKey())
        {
            trailEntry.PrimaryKey = property.CurrentValue?.ToString();
            continue;
        }

        // 过滤掉不应出现在审计列表中的属性
        if (property.Metadata.Name.Equals("PasswordHash"))
        {
            continue;
        }

        SetAuditTrailPropertyValue(entry, trailEntry, property);
    }
}

private static void SetAuditTrailPropertyValue(EntityEntry entry, AuditTrail trailEntry, PropertyEntry property)
{
    var propertyName = property.Metadata.Name;

    switch (entry.State)
    {
        case EntityState.Added:
            trailEntry.TrailType = TrailType.Create;
            trailEntry.NewValues[propertyName] = property.CurrentValue;

            break;

        case EntityState.Deleted:
            trailEntry.TrailType = TrailType.Delete;
            trailEntry.OldValues[propertyName] = property.OriginalValue;

            break;

        case EntityState.Modified:
            if (property.IsModified && (property.OriginalValue is null || !property.OriginalValue.Equals(property.CurrentValue)))
            {
                trailEntry.ChangedColumns.Add(propertyName);
                trailEntry.TrailType = TrailType.Update;
                trailEntry.OldValues[propertyName] = property.OriginalValue;
                trailEntry.NewValues[propertyName] = property.CurrentValue;
            }

            break;
    }

    if (trailEntry.ChangedColumns.Count > 0)
    {
        trailEntry.TrailType = TrailType.Update;
    }
}
```

如果你需要排除任何敏感字段，可以在此处进行。例如，我们正在将 PasswordHash 属性从审计追踪中排除。

现在，让我们探讨如何将**引用**和**导航**属性添加到审计追踪中：

```csharp
private static void SetAuditTrailReferenceValues(EntityEntry entry, AuditTrail trailEntry)
{
    foreach (var reference in entry.References.Where(x => x.IsModified))
    {
        var referenceName = reference.EntityEntry.Entity.GetType().Name;
        trailEntry.ChangedColumns.Add(referenceName);
    }
}

private static void SetAuditTrailNavigationValues(EntityEntry entry, AuditTrail trailEntry)
{
    foreach (var navigation in entry.Navigations.Where(x => x.Metadata.IsCollection && x.IsModified))
    {
        if (navigation.CurrentValue is not IEnumerable<object> enumerable)
        {
            continue;
        }

        var collection = enumerable.ToList();
        if (collection.Count == 0)
        {
            continue;
        }

        var navigationName = collection.First().GetType().Name;
        trailEntry.ChangedColumns.Add(navigationName);
    }
}
```

最后，我们可以运行应用程序，查看审计功能。

以下是 authors 表中由系统和用户设置的审计属性示例：

![审计属性示例](https://antondevtips.com/media/code_screenshots/efcore/audit-trails/img_1.png)

audit_trails 表格如下所示：

![审计追踪示例](https://antondevtips.com/media/code_screenshots/efcore/audit-trails/img_2.png)

![审计追踪示例](https://antondevtips.com/media/code_screenshots/efcore/audit-trails/img_3.png)
