# [如何在 ASP.NET Core 中使用 EF Core 实现多租户](https://antondevtips.com/blog/how-to-implement-multitenancy-in-asp-net-core-with-ef-core)

多租户是一种软件架构，它允许单个软件应用程序实例服务多个客户，这些客户被称为租户。出于安全原因，每个租户的数据都是隔离的，并且对其他租户不可见。这种架构常用于软件即服务（SaaS）应用程序中，其中多个组织或用户共享相同的应用程序基础设施，同时保持其数据安全和独立。

今天，我将与你分享我使用 Entity Framework Core (EF Core) 在 ASP.NET Core 中实现多租户的经验。

## 多租户简介

多租户具有以下几个优点：

- **成本效益**：共享基础设施降低了托管、基础设施和维护的总成本。
- **简化部署**：集中式更新简化了维护和支持。
- **可扩展性**：它允许根据每个租户的需求轻松扩展资源。
- **隔离和安全性**：每个租户的数据都是隔离的，确保了安全性和隐私。

在多租户应用程序中，有几种方法可以分离每个租户的数据：

- **每个租户一个数据库**：每个租户都有自己的数据库。这种模式提供了强大的数据隔离，但租户数量多时可能会增加成本。
- **每个租户一个结构纲目**：一个数据库，每个租户有独立的结构纲目。它在隔离和资源共享之间取得了平衡。
- **每个租户一张表**：一个数据库和一个结构纲目，包含租户特定的表。这种模式效率高，但可能会使数据管理复杂化。
- **判别列**：一个数据库、一个结构纲目和多张表，其中一列指示租户。这是最简单但隔离性最差的模式。

**判别列**是实现多租户最流行的方法之一。与其他选项相比，它在开发、部署和管理方面成本较低。借助 ASP.NET Core 和 EF Core 等现代技术，你可以在应用程序中实现判别列，而不会对性能和安全性产生负面影响。

接下来，我将向你展示如何使用判别列实现多租户。让我们探索一个需要多租户的应用程序。

## 多租户应用程序示例

今天我们将为“图书”应用程序实现多租户，该应用程序包含以下实体：

- Books 书籍
- Authors 作者
- Users 用户
- Tenants 租户

**Tenant** 实体保存了每个客户的信息：

```csharp
public class Tenant : IAuditableEntity
{
    public required Guid Id { get; set; }
    public required string Name { get; set; }
    
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
}
```

我们的数据库中，每个 Book 、 Author 和 User 实体都属于一个特定的租户。你需要创建一个 ITenantEntity 接口，所有需要多租户的实体都应继承该接口：

```csharp
public interface ITenantEntity
{
    public Guid? TenantId { get; set; }
}
```

我们以 User 和 Book 实体为例：

```csharp
public class User : IAuditableEntity, ITenantEntity
{
    public Guid Id { get; set; }
    public required string Email { get; set; }
    
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    
    public Guid? TenantId { get; set; }
}

public class Book : IAuditableEntity, ITenantEntity
{
    public required Guid Id { get; set; }
    public required string Title { get; set; }
    public required int Year { get; set; }
    public Guid AuthorId { get; set; }
    public Author Author { get; set; } = null!;

    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }

    public Guid? TenantId { get; set; }
}
```

每个实体都与 Tenant 实体存在外键关系，例如 User：

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("users");

        builder.HasKey(x => x.Id);
        builder.HasIndex(x => x.Email);

        builder.Property(x => x.Id).ValueGeneratedOnAdd();
        builder.Property(x => x.Email).IsRequired();
        builder.Property(x => x.CreatedAtUtc).IsRequired();
        builder.Property(x => x.UpdatedAtUtc);

        builder.HasOne<Tenant>()
            .WithMany()
            .HasForeignKey(e => e.TenantId)
            .IsRequired(false)
            .OnDelete(DeleteBehavior.SetNull);
    }
}
```

根据你的应用程序需求，你可以选择使用这个外键，或者使用一个不带引用的普通 TenantId 列。

## 如何在 EF Core 中实现多租户

要实现多租户，我们需要完成以下工作：

1. 用户登录后，我们需要在用户声明中添加租户标识符。
2. 前端（或其他应用程序）发出的每个请求，无论是检索还是修改数据，我们都需要从用户声明中获取用户租户标识符。
3. 每当用户从后端请求数据时，我们可以使用 EF Core 全局查询过滤器自动从数据库中筛选出用户有权访问的记录。
4. 每当用户创建、更新或删除数据时，我们可以在更改跟踪器中自动为“TenantId”列分配一个用户租户标识符。

通过利用 EF Core 的功能（全局查询过滤器和更改跟踪器），我们不再需要在每个查询中编写自定义过滤器，也不需要在每个创建、更新或删除操作中提供租户 ID。EF Core 帮助我们实现一次多租户，该实现将应用于所有需要多租户的实体。

这种方法可以保护你的代码免受关键错误的困扰，例如你可能会忘记在某个查询中添加租户检查，从而暴露其他客户的数据。

## 在 EF Core 中为读取操作实现多租户

我们可以在 EF Core DbContext 中实现多租户，它将自动应用于所有继承自 ITenantEntity 的实体。

让我们定义一个 **TenantProvider** ，它将从当前的 HttpRequest 中检索用户和租户标识符：

```csharp
public interface ITenantProvider
{
    TenantInfo GetCurrentTenantInfo();
}

public class TenantProvider : ITenantProvider
{
    private readonly TenantInfo _tenantInfo;

    public TenantProvider(IHttpContextAccessor accessor)
    {
        var userIdValue = accessor.HttpContext?.User.FindFirstValue("user-id");
        var tenantIdValue = accessor.HttpContext?.User.FindFirstValue("tenant-id");

        Guid? userId = Guid.TryParse(userIdValue, out var guid) ? guid : null;
        Guid? tenantId = Guid.TryParse(tenantIdValue, out guid) ? guid : null;

        _tenantInfo = new TenantInfo(userId, tenantId);
    }

    public TenantInfo GetCurrentTenantInfo() => _tenantInfo;
}
```

我们从 ClaimsPrinciple 中检索“用户 ID”和“租户 ID”。

你需要在 DI 中注册提供程序和 **IHttpContextAccessor** ：

```csharp
services.AddHttpContextAccessor();
services.AddScoped<ITenantProvider, TenantProvider>();
```

我们需要将 **ITenantProvider** 注入到 DbContext 中并为其创建一个公共属性：

```csharp
public class ApplicationDbContext(
    DbContextOptions<ApplicationDbContext> options,
    ITenantProvider tenantProvider)
    : DbContext(options)
{
    public ITenantProvider TenantProvider => tenantProvider;

    public DbSet<Author> Authors { get; set; } = default!;
    public DbSet<Book> Books { get; set; } = default!;
    public DbSet<User> Users { get; set; } = default!;
    public DbSet<Tenant> Tenants { get; set; } = default!;
}
```

在 **OnModelCreating** 方法中，你可以为所有租户实体指定全局查询过滤器：

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>()
        .HasQueryFilter(x => x.TenantId.Equals(TenantProvider.GetCurrentTenantInfo().TenantId));

    modelBuilder.Entity<Author>()
        .HasQueryFilter(x => x.TenantId.Equals(TenantProvider.GetCurrentTenantInfo().TenantId));

    modelBuilder.Entity<Book>()
        .HasQueryFilter(x => x.TenantId.Equals(TenantProvider.GetCurrentTenantInfo().TenantId));

    base.OnModelCreating(modelBuilder);

    modelBuilder.HasDefaultSchema("devtips_multitenancy");
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(BookConfiguration).Assembly);
}
```

在调用 **base.OnModelCreating(modelBuilder);** 之前，指定 **HasQueryFilter** 很重要。

每当你在项目中添加一个新实体时，只需添加一个新的查询过滤器，该实体的所有读取操作都将根据用户的租户进行相应过滤。这确保了用户只能访问他们自己的数据。

> 使用带有公共属性的 TenantProvider 至关重要，否则 EF Core 全局查询过滤器将无法正确应用于每个请求。

## 在 EF Core 中实现写入操作的多租户

我们可以在 EF Core DbContext 中重写 SaveChangesAsync 方法，以实现写入操作的多租户功能：

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new())
{
    var tenantInfo = TenantProvider.GetCurrentTenantInfo();

    var modifiedTenantEntries = ChangeTracker.Entries<ITenantEntity>()
        .Where(x => x.State is EntityState.Added or EntityState.Modified);

    foreach (var entry in modifiedTenantEntries)
    {
        entry.Entity.TenantId = tenantInfo.TenantId
            ?? throw new InvalidOperationException($"Tenant id is required but was not provided for entity '{entry.Entity.GetType()}' with state '{entry.State}'");
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```

我们遍历租户实体列表并自动设置 TenantId 属性。如果由于任何原因租户标识符不可用，则会抛出异常并中止操作。每当我们创建、更新或删除实体时，都必须设置租户标识符。

现在，让我们来了解多租户的实际应用。

## 在登录时向用户声明添加租户标识符

当用户登录时，我们需要使用一个 **IgnoreQueryFilters** 方法来在每个租户中搜索用户：

```csharp
public class LoginUserEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/users/login", Handle);
    }

    private static async Task<IResult> Handle(
        [FromBody] LoginUserRequest request,
        ApplicationDbContext context,
        IOptions<AuthConfiguration> jwtSettingsOptions,
        CancellationToken cancellationToken)
    {
        var user = await context.Users
            .IgnoreQueryFilters()
            .FirstOrDefaultAsync(u => u.Email == request.Email, cancellationToken);

        if (user is null)
        {
            return Results.NotFound("User not found");
        }

        var token = GenerateJwtToken(user, jwtSettingsOptions.Value);
        return Results.Ok(new { Token = token });
    }
}
```

此方法禁用应用于 User 实体的所有查询筛选器。找到用户后，你需要添加一个“租户 ID”声明：

```csharp
private static string GenerateJwtToken(User user, AuthConfiguration auth)
{
    var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(auth.Key));
    var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

    var claims = new[]
    {
        new Claim(JwtRegisteredClaimNames.Sub, user.Email),
        new Claim("use-id", user.Id.ToString()),
        new Claim("tenant-id", user.TenantId?.ToString() ?? string.Empty)
    };

    var token = new JwtSecurityToken(
        issuer: auth.Issuer,
        audience: auth.Audience,
        claims: claims,
        expires: DateTime.Now.AddMinutes(30),
        signingCredentials: credentials
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

用户登录后，我们就可以开始使用 JWT 令牌调用其他端点了。

## 为多租户实体实现 API 端点

让我们为图书实体探索 Create 和 Get By Id 端点：

```csharp
public sealed record CreateBookRequest(string Title, int Year, Guid AuthorId);

public class CreateBookEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/books", Handle);
    }

    private static async Task<IResult> Handle(
        [FromBody] CreateBookRequest request,
        ApplicationDbContext context,
        CancellationToken cancellationToken)
    {
        var author = await context.Authors.FindAsync([request.AuthorId], cancellationToken);
        if (author is null)
        {
            return Results.BadRequest("Author not found");
        }

        var book = new Book
        {
            Id = Guid.NewGuid(),
            Title = request.Title,
            Year = request.Year,
            AuthorId = request.AuthorId,
            Author = author
        };

        context.Books.Add(book);
        await context.SaveChangesAsync(cancellationToken);

        var response = new BookResponse(book.Id, book.Title, book.Year, book.AuthorId);
        return Results.Created($"/api/books/{book.Id}", response);
    }
}
```

```csharp
public class GetBookByIdEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/books/{id}", Handle);
    }

    private static async Task<IResult> Handle(
        [FromRoute] Guid id,
        ApplicationDbContext context,
        CancellationToken cancellationToken)
    {
        var book = await context.Books
            .Include(b => b.Author)
            .FirstOrDefaultAsync(b => b.Id == id, cancellationToken);

        if (book is null)
        {
            return Results.NotFound();
        }

        var response = new BookResponse(book.Id, book.Title, book.Year, book.AuthorId);
        return Results.Ok(response);
    }
}
```

正如你可能已经预料到的，代码对租户一无所知。而这正是我们所追求的——创建一个安全的实现，它现在将允许客户与另一个客户的数据进行交互，而无需在每次数据库调用时都担心租户问题。

创建图书时，我们会根据请求中提供的 AuthorId 检查数据库中是否存在作者。EF Core 确保我们不能使用来自其他租户的作者添加图书。嗯，这太棒了！

## 使用租户 ID 请求头

在某些应用程序中，一个用户可以访问多个租户，例如，“超级管理员”用户可以管理不同租户的实体。在这种情况下，每当我们需要检索或修改数据时，都需要从前端（或其他应用程序）的每个请求中发送“X-TenantId”请求头。

如果你有这样的情况，你可以修改你的 TenantProvider 如下：

```csharp
public class TenantProvider : ITenantProvider
{
    private readonly TenantInfo _tenantInfo;

    public TenantProvider(IHttpContextAccessor accessor)
    {
        var userIdValue = accessor.HttpContext?.User.FindFirstValue("user-id");

        Guid? userId = Guid.TryParse(userIdValue, out var guid) ? guid : null;
        Guid? headerTenantId = null;

        if (accessor.HttpContext?.Request.Headers.TryGetValue("X-TenantId", out var headerGuid) is true)
        {
            headerTenantId = Guid.Parse(headerGuid.ToString());
        }

        _tenantInfo = new TenantInfo(userId, headerTenantId);
    }

    public TenantInfo GetCurrentTenantInfo() => _tenantInfo;
}
```

在某些应用程序中，最好始终使用“X-TenantId”标头，而不是将其放在 JWT 令牌中。当你使用对租户一无所知的外部身份验证提供程序时，也可能出现这种情况。

解决这个问题有不同的方法。我将展示一种可能的解决方案，使用一个中间件来检查用户是否拥有访问“X-TenantId”的权限。如果用户无权访问所请求的租户，则返回 403 Forbidden 响应。

```csharp
public class TenantCheckerMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var tenantClaimValue = context.User.Claims.FirstOrDefault(x => x.Type.Equals("tenant-id"))?.Value;
        if (tenantClaimValue is null)
        {
            await next(context);
            return;
        }

        if (!context.Request.Headers.TryGetValue("X-TenantId", out var headerGuid))
        {
            await next(context);
            return;
        }

        if (tenantClaimValue.Contains(headerGuid.ToString(), StringComparison.Ordinal))
        {
            await next(context);
            return;
        }

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status403Forbidden,
            Title = "Bad Request",
            Detail = "X-TenantId header contains a tenant id that a user doesn't have access to"
        };

        context.Response.StatusCode = problemDetails.Status.Value;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problemDetails);
    }
}
```

在这个例子中，我做了一个基本检查，看“X-TenantId”头是否与 JWT 令牌中的“tenant-id”匹配。在你的情况下，你可能想在数据库中检查，或者更好是在缓存中检查。

## 条件式全局查询筛选器

遗憾的是，EF Core 不支持带条件的全局查询筛选器，尽管这是一个呼声很高的功能。

例如，你可能希望实现以下查询筛选器，该筛选器仅在用户对租户的访问权限受限时应用。而“超级管理员”用户可以查看所有租户。

```csharp
if (!TenantProvider.GetCurrentTenantInfo().IsSuperAdmin)
{
    builder.Entity<User>()
        .HasQueryFilter(x => x.TenantId.Equals(TenantProvider.GetCurrentTenantInfo().TenantId));
}
```

如果你尝试测试这个应用程序，即使是普通用户，你的过滤器也不会触发。所有这些都是因为 DbContext OnModelCreating 方法在应用程序的整个生命周期中只被调用一次——当第一个 DbContext 实例被创建时。

如果确实需要使用这种根据每个请求有条件应用的全局查询筛选器，可以创建一个 **DynamicModelCacheKeyFactory** ：

```csharp
public class DynamicModelCacheKeyFactory : IModelCacheKeyFactory
{
    public object Create(DbContext context, bool designTime)
        => context is ApplicationDbContext dynamicContext
            ? (context.GetType(), dynamicContext.TenantProvider.GetCurrentTenantInfo(), designTime)
            : context.GetType();
    
    public object Create(DbContext context)
        => Create(context, false);
```

注册 DbContext 时，你需要用动态的 **IModelCacheKeyFactory** 替换标准的 **IModelCacheKeyFactory** ：

```csharp
builder.Services.AddDbContext<ApplicationDbContext>((provider, options) =>
{
    var interceptor = provider.GetRequiredService<AuditableInterceptor>();

    options.EnableSensitiveDataLogging()
        .UseNpgsql(connectionString, npgsqlOptions =>
        {
            npgsqlOptions.MigrationsHistoryTable("__MyMigrationsHistory", "devtips_multitenancy");
        })
        .AddInterceptors(interceptor)
        .UseSnakeCaseNamingConvention();
        
    options.ReplaceService<IModelCacheKeyFactory, DynamicModelCacheKeyFactory>();
});
```

这个 **DynamicModelCacheKeyFactory** 告诉 EF Core 不要缓存已创建的模型（与查询过滤器映射），并为 DbContext 的每个实例调用 OnModelCreating 方法。这确保了条件全局查询过滤器能够工作，但这种方法会严重损害 DbContext 中每次数据库调用的性能。因此，请谨慎使用并对你的请求进行基准测试。

今天就到这里。期待与你再次相见。
