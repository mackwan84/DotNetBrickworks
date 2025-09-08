# [如何使用编译查询优化 EF Core 查询性能](https://medium.com/@michaelmaurice410/how-to-optimize-ef-core-query-performance-with-compiled-queries-a50537f3695e)

Entity Framework Core 的编译查询是 .NET 开发者可用的但又最未被充分利用的性能优化功能之一。虽然 EF Core 会自动缓存查询，但编译查询通过完全消除查询编译开销，将优化提升到了一个新水平。今天，我将向你展示如何实现编译查询，并在数据访问层实现 10-40%的性能提升。

## 理解 EF Core 中的查询编译

### 数据查询的生命周期问题

当 EF Core 执行 LINQ 查询时，它会经历几个耗时的步骤：

```csharp
// 每次查询执行时发生的事情
var user = await context.Users
    .Where(u => u.Id == userId)
    .FirstOrDefaultAsync();
// EF Core 执行步骤：
// 1. 解析 LINQ 表达式树
// 2. 与缓存的查询进行比较
// 3. 转换为 SQL（如果未缓存）
// 4. 执行数据库操作
// 5. 将结果物化回对象
```

性能开销：

- 表达式树分析：EF Core 必须递归地比较表达式树
- 缓存查找：找到正确的缓存查询需要消耗 CPU 周期
- 翻译过程：将 LINQ 转换为 SQL 需要计算
- 内存分配：在翻译过程中创建临时对象

### 编译查询如何解决这个问题

编译查询将 LINQ 表达式预编译为可重用的委托，从而完全消除了步骤 1-3：

```csharp
// 当程序启动时只需编译一次
private static readonly Func<ApplicationDbContext, int, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id) => 
        context.Users.FirstOrDefault(u => u.Id == id));
// 直接执行 - 无需编译开销
var user = await GetUserById(context, userId);
```

这么做的好处：

- 消除编译：无需表达式树解析或转换
- 直接执行：无需缓存查找，立即执行 SQL
- 一致的性能：可预测的执行时间
- 内存效率：减少临时对象分配

## 基本编译查询实现

### 简单实体检索

从常见操作的基本编译查询开始：

```csharp
// Data/CompiledQueries.cs
using Microsoft.EntityFrameworkCore;
namespace MyApp.Data;
public static class CompiledQueries
{
    // 同步方式的编译查询
    public static readonly Func<ApplicationDbContext, int, User?> GetUserById =
        EF.CompileQuery((ApplicationDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
    // 异步方式的编译查询（推荐用于大多数场景）
    public static readonly Func<ApplicationDbContext, int, Task<User?>> GetUserByIdAsync =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
    // 带有多个参数的查询
    public static readonly Func<ApplicationDbContext, string, string, Task<User?>> GetUserByEmailAndStatus =
        EF.CompileAsyncQuery((ApplicationDbContext context, string email, string status) =>
            context.Users.FirstOrDefault(u => u.Email == email && u.Status == status));
    // 带有包含的查询
    public static readonly Func<ApplicationDbContext, int, Task<User?>> GetUserWithOrders =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users
                .Include(u => u.Orders)
                .FirstOrDefault(u => u.Id == id));
    // 针对只读场景的、不带跟踪的查询
    public static readonly Func<ApplicationDbContext, int, Task<User?>> GetUserByIdNoTracking =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users
                .AsNoTracking()
                .FirstOrDefault(u => u.Id == id));
}
```

### 仓储集成

将编译查询集成到你的仓储模式中：

```csharp
// Repositories/UserRepository.cs
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    // 直接使用编译查询
    public async Task<User?> GetByIdAsync(int id)
    {
        return await CompiledQueries.GetUserByIdAsync(_context, id);
    }
    public async Task<User?> GetByEmailAndStatusAsync(string email, string status)
    {
        return await CompiledQueries.GetUserByEmailAndStatus(_context, email, status);
    }
    public async Task<User?> GetWithOrdersAsync(int id)
    {
        return await CompiledQueries.GetUserWithOrders(_context, id);
    }
    // 获得更好性能的只读查询
    public async Task<User?> GetByIdReadOnlyAsync(int id)
    {
        return await CompiledQueries.GetUserByIdNoTracking(_context, id);
    }
}
```

## 高级编译查询模式

### 复杂筛选和聚合

处理更复杂的查询场景：

```csharp
// Data/AdvancedCompiledQueries.cs
public static class AdvancedCompiledQueries
{
    // 多个条件的复杂筛选
    public static readonly Func<ApplicationDbContext, DateTime, string, int, Task<IEnumerable<Order>>> GetOrdersByDateRangeAndStatus =
        EF.CompileAsyncQuery((ApplicationDbContext context, DateTime fromDate, string status, int customerId) =>
            context.Orders
                .Where(o => o.CreatedDate >= fromDate && 
                           o.Status == status && 
                           o.CustomerId == customerId)
                .OrderByDescending(o => o.CreatedDate)
                .AsEnumerable()); // 注意：返回 IEnumerable 以供编译查询使用
    // 聚合查询
    public static readonly Func<ApplicationDbContext, int, Task<decimal>> GetCustomerTotalSpent =
        EF.CompileAsyncQuery((ApplicationDbContext context, int customerId) =>
            context.Orders
                .Where(o => o.CustomerId == customerId)
                .Sum(o => o.TotalAmount));
    // 计数查询
    public static readonly Func<ApplicationDbContext, string, Task<int>> GetActiveUserCount =
        EF.CompileAsyncQuery((ApplicationDbContext context, string status) =>
            context.Users.Count(u => u.Status == status));
    // 映射查询
    public static readonly Func<ApplicationDbContext, int, Task<UserSummaryDto?>> GetUserSummary =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users
                .Where(u => u.Id == id)
                .Select(u => new UserSummaryDto
                {
                    Id = u.Id,
                    Name = u.FirstName + " " + u.LastName,
                    Email = u.Email,
                    OrderCount = u.Orders.Count(),
                    TotalSpent = u.Orders.Sum(o => o.TotalAmount)
                })
                .FirstOrDefault());
    // 参数化日期范围查询
    public static readonly Func<ApplicationDbContext, DateTime, DateTime, Task<IEnumerable<Order>>> GetOrdersInDateRange =
        EF.CompileAsyncQuery((ApplicationDbContext context, DateTime startDate, DateTime endDate) =>
            context.Orders
                .Where(o => o.CreatedDate >= startDate && o.CreatedDate <= endDate)
                .Include(o => o.Customer)
                .AsEnumerable());
}

public record UserSummaryDto
{
    public int Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;
    public int OrderCount { get; init; }
    public decimal TotalSpent { get; init; }
}
```

### 处理列表返回

解决编译查询在返回集合方面的限制：

```csharp
// 困境：编译查询无法直接返回 List<T>
// 解决方案：返回 IEnumerable<T>，并在需要时转换为 List<T>
public static class CollectionCompiledQueries
{
    // 从编译查询中返回 IEnumerable
    public static readonly Func<ApplicationDbContext, string, IEnumerable<User>> GetUsersByStatus =
        EF.CompileQuery((ApplicationDbContext context, string status) =>
            context.Users.Where(u => u.Status == status).AsEnumerable());
    // 对于异步集合，使用 IAsyncEnumerable<T>
    public static readonly Func<ApplicationDbContext, string, IAsyncEnumerable<User>> GetUsersByStatusAsync =
        EF.CompileAsyncQuery((ApplicationDbContext context, string status) =>
            context.Users.Where(u => u.Status == status));
    // 转换为 List 的仓储方法
    public async Task<List<User>> GetActiveUsersListAsync(string status)
    {
        var asyncEnumerable = GetUsersByStatusAsync(_context, status);
        var result = new List<User>();
        
        await foreach (var user in asyncEnumerable)
        {
            result.Add(user);
        }
        
        return result;
    }
    // 另一种选择：使用 ToListAsync() 扩展方法
    public async Task<List<User>> GetActiveUsersListAlternativeAsync(string status)
    {
        var asyncEnumerable = GetUsersByStatusAsync(_context, status);
        return await asyncEnumerable.ToListAsync();
    }
}
```

## DbContext 集成

### DbContext 级别编译查询

将编译查询直接集成到你的 DbContext 中：

```csharp
// Data/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<Product> Products { get; set; }
    // 将编译查询定义为静态只读字段
    private static readonly Func<ApplicationDbContext, int, Task<User?>> _getUserByIdQuery =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
    private static readonly Func<ApplicationDbContext, string, Task<IEnumerable<User>>> _getUsersByStatusQuery =
        EF.CompileAsyncQuery((ApplicationDbContext context, string status) =>
            context.Users.Where(u => u.Status == status));
    private static readonly Func<ApplicationDbContext, int, Task<decimal>> _getCustomerTotalQuery =
        EF.CompileAsyncQuery((ApplicationDbContext context, int customerId) =>
            context.Orders
                .Where(o => o.CustomerId == customerId)
                .Sum(o => o.TotalAmount));
    // 封装编译查询
    public async Task<User?> GetUserByIdCompiledAsync(int id)
    {
        return await _getUserByIdQuery(this, id);
    }
    public async Task<IEnumerable<User>> GetUsersByStatusCompiledAsync(string status)
    {
        return await _getUsersByStatusQuery(this, status);
    }
    public async Task<decimal> GetCustomerTotalCompiledAsync(int customerId)
    {
        return await _getCustomerTotalQuery(this, customerId);
    }
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
    {
    }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
    }
}
```

## 使用 DbContext 池以实现最大性能

将编译查询与 DbContext 池结合使用以获得最佳性能：

```csharp
/ Program.cs - Configure DbContext pooling
var builder = WebApplication.CreateBuilder(args);
// 使用 DbContextPool 代替 AddDbContext 以获得更好的编译查询性能
builder.Services.AddDbContextPool<ApplicationDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));

    options.EnableServiceProviderCaching();
    options.EnableSensitiveDataLogging(builder.Environment.IsDevelopment());
}, poolSize: 128); // 可选：配置池大小（默认值为 1024）
var app = builder.Build();
// 在控制器/服务中使用时会自动受益于池化
public class UserController : ControllerBase
{
    private readonly ApplicationDbContext _context;
    public UserController(ApplicationDbContext context)
    {
        _context = context; // 这是来自池的实例
    }
    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetUser(int id)
    {
        // 使用编译查询 + 池化上下文以获得最佳性能
        var user = await _context.GetUserByIdCompiledAsync(id);
        return user == null ? NotFound() : Ok(user);
    }
}
```

## 性能基准测试

### 性能对比分析

真实基准测试结果显示性能显著提升：

```csharp
// Benchmarks/QueryPerformanceBenchmark.cs
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
[MemoryDiagnoser]
[SimpleJob(BenchmarkDotNet.Jobs.RuntimeMoniker.Net80)]
public class QueryPerformanceBenchmark
{
    private ApplicationDbContext _context = null!;
    private const int TestUserId = 7000;
    [GlobalSetup]
    public void Setup()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=PerformanceTest;Trusted_Connection=true")
            .Options;
            
        _context = new ApplicationDbContext(options);
        
        // Ensure database exists and is seeded
        _context.Database.EnsureCreated();
        SeedTestData();
    }
    [Benchmark(Baseline = true)]
    public async Task<User?> RegularQuery()
    {
        return await _context.Users.FirstOrDefaultAsync(u => u.Id == TestUserId);
    }
    [Benchmark]
    public async Task<User?> CompiledQuery()
    {
        return await CompiledQueries.GetUserByIdAsync(_context, TestUserId);
    }
    [Benchmark]
    public async Task<User?> CompiledQueryNoTracking()
    {
        return await CompiledQueries.GetUserByIdNoTracking(_context, TestUserId);
    }
    [Benchmark]
    public async Task<User?> RegularQueryWithIncludes()
    {
        return await _context.Users
            .Include(u => u.Orders)
            .FirstOrDefaultAsync(u => u.Id == TestUserId);
    }
    [Benchmark]
    public async Task<User?> CompiledQueryWithIncludes()
    {
        return await CompiledQueries.GetUserWithOrders(_context, TestUserId);
    }
    private void SeedTestData()
    {
        if (!_context.Users.Any())
        {
            var users = new List<User>();
            for (int i = 1; i <= 10000; i++)
            {
                users.Add(new User
                {
                    Id = i,
                    FirstName = $"User{i}",
                    LastName = $"LastName{i}",
                    Email = $"@example.com">user{i}@example.com",
                    Status = i % 2 == 0 ? "Active" : "Inactive"
                });
            }
            
            _context.Users.AddRange(users);
            _context.SaveChanges();
        }
    }
    [GlobalCleanup]
    public void Cleanup()
    {
        _context.Dispose();
    }
}
```

基准测试结果

|                Method |      Mean |     Error |    StdDev | Ratio |  Gen 0 | Allocated |
|---------------------- |----------:|----------:|----------:|------:|-------:|----------:|
|           RegularQuery | 284.5 μs |   5.2 μs |   4.9 μs |  1.00 | 0.4883 |    2048 B |
|         CompiledQuery | 245.8 μs |   3.1 μs |   2.9 μs |  0.86 | 0.4883 |    1856 B |
| CompiledQueryNoTracking | 198.2 μs |   2.8 μs |   2.6 μs |  0.70 | 0.2441 |    1024 B |

观察到的性能改进：

- 简单查询：10–15% 提升
- 复杂查询：5–10% 的性能提升
- 无跟踪查询：结合使用时可提升 30%性能
- 内存使用：分配减少 10–20%

## 最佳实践与指南

### 何时使用编译查询

✅ 完美使用场景：

- 频繁执行的查询：重复调用相同查询结构
- 性能关键路径：高流量应用中的热点代码路径
- 简单到中等复杂度：简单的 LINQ 表达式
- 稳定的查询模式：查询结构不经常变化

❌ 避免在以下情况使用：

- 动态查询：查询结构根据运行时条件变化
- 复杂的 LINQ：大量使用客户端评估或复杂映射
- 一次性查询：很少执行的查询不会从编译中受益
- 高度参数化：参数组合过多，不切实际

### 实现最佳实践

结构组织：

```csharp
// 将编译查询组织在专用静态类中
public static class UserCompiledQueries
{
    public static readonly Func<ApplicationDbContext, int, Task<User?>> GetById = /* ... */;
    public static readonly Func<ApplicationDbContext, string, Task<User?>> GetByEmail = /* ... */;
}
public static class OrderCompiledQueries  
{
    public static readonly Func<ApplicationDbContext, int, Task<Order?>> GetById = /* ... */;
    public static readonly Func<ApplicationDbContext, DateTime, Task<IEnumerable<Order>>> GetByDate = /* ... */;
}
```

参数优化：

```csharp
// 正确的用法：指定的、集中的参数
public static readonly Func<ApplicationDbContext, int, string, Task<User?>> GetUserByIdAndStatus =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id, string status) =>
        context.Users.FirstOrDefault(u => u.Id == id && u.Status == status));
// 尽量避免：过多的参数以致使缓存失效
public static readonly Func<ApplicationDbContext, int, string, string, DateTime, bool, Task<User?>> OverParameterized =
    EF.CompileAsyncQuery((ApplicationDbContext context, int id, string firstName, string lastName, DateTime birthDate, bool isActive) =>
        /* complex query */);
```

### 错误处理与调试

优雅地处理编译查询的限制：

```csharp
public class SafeCompiledQueryService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<SafeCompiledQueryService> _logger;

    public async Task<User?> GetUserSafelyAsync(int id)
    {
        try
        {
            return await CompiledQueries.GetUserByIdAsync(_context, id);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Compiled query failed for user {UserId}, falling back to regular query", id);
            
            // 如遇异常，则重新发起正常查询
            return await _context.Users.FirstOrDefaultAsync(u => u.Id == id);
        }
    }
    public async Task<List<User>> GetUsersWithFallbackAsync(string status)
    {
        try
        {
            var result = CompiledQueries.GetUsersByStatusAsync(_context, status);
            return await result.ToListAsync();
        }
        catch (NotSupportedException ex)
        {
            _logger.LogWarning("Compiled query not supported: {Message}, using regular query", ex.Message);
            
            return await _context.Users
                .Where(u => u.Status == status)
                .ToListAsync();
        }
    }
}
```

## 限制与注意事项

### 编译查询的局限性

技术限制：

- 表达式树限制：并非所有 LINQ 表达式都能被编译
- 返回类型限制：有限的返回类型（不能直接使用 List<T> ）
- 客户端评估：不能包含客户端逻辑
- 动态查询：不能用于动态构造的查询

针对以上限制的解决方法：

```csharp
// 编译限制：无法直接返回 List<T>
// 解决方法：返回 IAsyncEnumerable<T> 并转换
public async Task<List<User>> GetUsersAsListAsync(string status)
{
    var asyncEnumerable = CompiledQueries.GetUsersByStatusAsync(_context, status);
    var users = new List<User>();
    
    await foreach (var user in asyncEnumerable)
    {
        users.Add(user);
    }
    
    return users;
}
// 另一种方案：使用扩展方法
public async Task<List<User>> GetUsersAsListAlternativeAsync(string status)
{
    return await CompiledQueries.GetUsersByStatusAsync(_context, status).ToListAsync();
}
```

### 内存与缓存考量

静态字段管理：

```csharp
// 正确方法：使用静态字段
public static class CompiledQueries
{
    // 定义一次，重复使用
    private static readonly Func<ApplicationDbContext, int, Task<User?>> _getUserById =
        EF.CompileAsyncQuery((ApplicationDbContext context, int id) =>
            context.Users.FirstOrDefault(u => u.Id == id));
    public static Task<User?> GetUserByIdAsync(ApplicationDbContext context, int id) =>
        _getUserById(context, id);
}
```

## 生产部署策略

### 渐进式推出方法

增量实现编译查询：

```csharp
// 第一阶段：只针对最关键的查询
public class UserService
{
    private readonly ApplicationDbContext _context;
    private readonly IConfiguration _configuration;
    public async Task<User?> GetUserAsync(int id)
    {
        // Feature flag for gradual rollout
        var useCompiledQueries = _configuration.GetValue<bool>("Features:UseCompiledQueries", false);
        
        return useCompiledQueries 
            ? await CompiledQueries.GetUserByIdAsync(_context, id)
            : await _context.Users.FirstOrDefaultAsync(u => u.Id == id);
    }
}
// 第二阶段：监测性能并扩大使用
// 第三阶段：验证后全面部署
```

## 监控与指标

跟踪性能改进：

```csharp
public class InstrumentedCompiledQueryService
{
    private readonly ApplicationDbContext _context;
    private readonly IMetrics _metrics;
    public async Task<User?> GetUserWithMetricsAsync(int id)
    {
        using var activity = Activity.StartActivity("GetUser");
        var stopwatch = Stopwatch.StartNew();
        try
        {
            var user = await CompiledQueries.GetUserByIdAsync(_context, id);
            
            stopwatch.Stop();
            _metrics.CreateHistogram<double>("user_query_duration_ms")
                .Record(stopwatch.Elapsed.TotalMilliseconds, 
                    new TagList { { "query_type", "compiled" } });
            return user;
        }
        catch (Exception ex)
        {
            _metrics.CreateCounter<long>("user_query_errors")
                .Add(1, new TagList { { "query_type", "compiled" } });
            throw;
        }
    }
}
```

## 总结

EF Core 编译查询为具有可预测查询模式和高频数据访问的应用程序提供了显著的性能改进：

主要优势：

- 10–40% 性能提升：查询执行时间可测量的减少
- 减少内存分配：通过优化执行路径降低 GC 压力
- 可预测的性能：一致的执行时间，无编译开销
- 更好的可扩展性：在高负载条件下提高吞吐量

实现策略：

- 识别候选查询：专注于频繁执行且稳定的查询模式
- 从简单开始：从基本的实体检索查询开始
- 衡量影响：使用基准测试来验证性能改进
- 结合池化：使用 DbContext 池化以获得最大收益
- 监控生产环境：跟踪性能指标和错误率

何时选择编译查询：

- 每分钟执行数千次的高频查询
- 对性能要求极高的应用程序，每一毫秒都至关重要
- 稳定的查询模式，不经常变化
- 简单到中等复杂度的 LINQ，无需大量客户端评估

编译查询是提升 EF Core 应用程序性能的最有效方法之一。虽然它们需要前期规划且存在一些限制，但 10-40%的性能提升和减少的内存分配使它们成为高性能.NET 应用程序的宝贵工具。结合其他 EF Core 优化技术（如 DbContext 池和 AsNoTracking()），编译查询有助于在保持 ORM 优势的同时，缩小 EF Core 与原始 SQL 之间的性能差距。

今天就到这里。期待与你再次相见。
