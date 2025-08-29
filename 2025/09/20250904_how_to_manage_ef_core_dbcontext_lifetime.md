# [如何管理 EF Core DbContext 的生命周期](https://antondevtips.com/blog/how-to-manage-ef-core-dbcontext-lifetime)

正确管理 DbContext 的生命周期对应用性能和稳定性至关重要。

在许多情况下，将 DbContext 注册为作用域范围（scoped）就足够简单，但也有需要更多控制的场景。EF Core 在 DbContext 创建方面提供了更大的灵活性。

今天，我将演示如何使用 DbContext、DbContextFactory 及它们的池化版本。这些功能允许更大的灵活性和更高的效率，特别是在需要高性能或具有特定线程模型的应用程序中。

## 使用 DbContext

DbContext 是 EF Core 的核心，它与数据库建立连接并允许执行增删改查（CRUD）操作。

DbContext 类负责：

- 管理数据库连接：根据需要打开和关闭与数据库的连接。
- 跟踪更改：跟踪对实体所做的更改，以便将它们持久化到数据库。
- 执行查询：将 LINQ 查询翻译为 SQL 并在数据库上执行。

在使用 DbContext 时，你应注意以下细节：

- 非线程安全：不应在多个线程上同时共享。
- 轻量级：设计为可频繁创建和释放。
- 有状态：跟踪实体状态以进行更改跟踪和身份解析。

让我们来探讨如何在依赖注入容器中注册 DbContext：

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
});

public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
    }
}
```

DbContext 在依赖注入容器中注册为 Scoped（作用域）。它的生存期为当前作用域，即与当前请求的持续时间相同：

```csharp
app.MapPost("/api/authors", async (
    [FromBody] CreateAuthorRequest request,
    ApplicationDbContext context,
    CancellationToken cancellationToken) =>
{
    var author = request.MapToEntity();

    context.Authors.Add(author);
    await context.SaveChangesAsync(cancellationToken);

    var response = author.MapToResponse();
    return Results.Created($"/api/authors/{author.Id}", response);
});
```

你也可以手动创建一个作用域并解析 DbContext：

```csharp
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await dbContext.Database.MigrateAsync();
}
```

将 DbContext 作为 Scoped 依赖项非常灵活，你可以将相同的 DbContext 实例注入到控制器/Minimal API 端点、服务、存储库中。但有时你需要更精确地控制何时创建和释放 DbContext 。通过 DbContextFactory 可以获得这样的控制。

## 使用 DbContextFactory

在某些使用场景中，例如后台服务、多线程应用程序或用于创建服务的工厂，你可能需要完全控制以创建和释放 DbContext 实例。

**IDbContextFactory\<TContext\>** 是 EF Core 提供的一项服务，允许按需创建 DbContext 实例。它确保每个实例都被正确配置，并且可以安全使用，而不直接与依赖注入容器的服务生命周期绑定。

你可以像注册 DbContext 那样以以下方式注册 **IDbContextFactory\<TContext\>** ：

```csharp
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
});
```

**IDbContextFactory** 在依赖注入容器中被注册为 **Singleton**。

你可以从 **IDbContextFactory** 使用 **CreateDbContext** 或 **CreateDbContextAsync** 来创建 DbContext：

```csharp
public class HostedService : IHostedService
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public HostedService(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        await using var context = await _contextFactory.CreateDbContextAsync(cancellationToken);
        var books = await context.Books.ToListAsync(cancellationToken: cancellationToken);
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

DbContext 在 using 块结束时被释放。

你在使用 **IDbContextFactory\<TContext\>** 时应谨慎——确保不会创建过多的数据库连接。

你可以同时注册 DbContext 和 IDbContextFactory 。需要做一个小调整，将 **optionsLifetime** 设置为 **Singleton** ，因为 IDbContextFactory 是以 **Singleton** 注册的：

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
}, optionsLifetime: ServiceLifetime.Singleton);

builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
});
```

## 使用 DbContext 池

DbContext 池是 EF Core 引入的一项功能，允许重用 DbContext 实例，从而减少频繁创建和释放上下文的开销。

DbContext 池维护一组预先配置好的 DbContext 实例。当你请求一个 DbContext 时，它会从池中提供一个。使用完毕后，它会重置该实例的状态并将其返回到池中以供重用。

此功能对于高性能场景或需要创建大量数据库连接时至关重要。

注册 DbContext 池很简单：

```csharp
builder.Services.AddDbContextPool<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
});
```

在你的类中，你注入了一个普通的 DbContext ，却不知道它被放入了池中。

## 使用 DbContextFactory 池

你可以将 DbContextFactory 和 DbContext 池结合起来，创建一个 DbContextFactory 池。这样可以按需创建 DbContext 实例，同时为了性能这些实例也会被池化。

```csharp
builder.Services.AddPooledDbContextFactory<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseNpgsql(connectionString);
});
```

使用工厂池的 API 是相同的：

```csharp
await using var context = await _contextFactory.CreateDbContextAsync(cancellationToken);
var books = await context.Books.ToListAsync(cancellationToken: cancellationToken);
```

每次 CreateDbContextAsync 调用都会从池中检索一个 DbContext。释放后，DbContext 会被返回到池中。

在你的类中，你注入了一个普通的 DbContextFactory ，却不知道它被放入了池中。

## 总结

管理 DbContext 生命周期对于构建高效的 EF Core 应用至关重要。通过利用 DbContextFactory 池和 DbContext 池，你可以更好地控制上下文的创建并优化性能。

### 要点

- DbContextFactory：当你需要按需创建 DbContext 实例，尤其是在多线程或后台任务中时使用。
- DbContext 池：用于减少频繁创建和释放 DbContext 实例的开销。
- DbContextFactory 池：结合两者的功能，根据需要创建池化的 DbContext 实例。
- 始终释放 DbContext 实例：使用 using 语句或确保上下文被释放或返回到池中。
- 避免线程安全问题：不要在多个线程之间共享 DbContext 实例。

今天就到这里。期待与你再次相见。
