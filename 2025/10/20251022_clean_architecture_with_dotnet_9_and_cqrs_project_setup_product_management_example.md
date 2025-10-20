# [使用 .NET 9 和 CQRS 的整洁架构](https://medium.com/@mariammaurice/clean-architecture-with-net-9-and-cqrs-project-setup-product-management-example-488ef8bef5e5)

当我第一次尝试组织一个中等规模的 .NET API 时，我感到非常沮丧。项目变得混乱：控制器泄露了业务规则，EF 实体直接返回给客户端，功能逻辑分散在各层之间。修复一个 bug 可能会破坏不相关的代码。我需要一种能让我专注于功能而不是解开依赖关系的模式——因此我采用了整洁架构并将其与 CQRS（通过 MediatR）结合起来。这一改变不仅让我的代码更整洁——还改变了我设计系统的方式。

下面是一份实用的、易于复制的指南，介绍如何使用 .NET 9、整洁架构和 CQRS 设置产品管理 API。我将向你展示项目结构、示例 CreateProduct 命令和 GetProductById 查询的实现、与 EF Core 的集成，以及一些关于验证和测试的说明。

你将获得的内容

- 推荐的项目结构。
- dotnet 用于创建项目的 CLI 命令。
- 领域层、应用层（CQRS）、基础设施层（EF Core）和 API 层（Web）的实现。
- 使用 MediatR 处理程序的命令（CreateProduct）和查询（GetProductById）示例。
- 仓储模式接口 + EF 实现。
- 在 Program.cs 中的依赖注入。
- 关于验证、日志记录、迁移和测试的说明。

## 先决条件

- 已安装 .NET 9 SDK。
- 代码编辑器（VS Code、Rider、Visual Studio）。
- SQL Server、SQLite 或任何 EF Core 提供程序（示例中使用 SQLite 以简化操作）。
- C# 和 ASP.NET Core Web API 的基础知识。

## 解决方案与项目结构

创建一个解决方案和多个项目（每层一个）：

```
MyProductApp/
├─ src/
│  ├─ MyProductApp.Domain/        (plain C# project - domain entities & interfaces)
│  ├─ MyProductApp.Application/   (CQRS: commands, queries, DTOs, interfaces)
│  ├─ MyProductApp.Infrastructure/(EF Core, repo implementations, migrations)
│  └─ MyProductApp.Api/           (ASP.NET Core Web API - controllers)
└─ tests/
   ├─ MyProductApp.Application.Tests
   └─ MyProductApp.Infrastructure.Tests
```

通过以下命令创建解决方案和项目：

```bash
mkdir MyProductApp && cd MyProductApp
dotnet new sln -n MyProductApp
# domain
dotnet new classlib -n MyProductApp.Domain -f net9.0
# application
dotnet new classlib -n MyProductApp.Application -f net9.0
# infrastructure
dotnet new classlib -n MyProductApp.Infrastructure -f net9.0
# api
dotnet new webapi -n MyProductApp.Api -f net9.0
dotnet sln add src/MyProductApp.Domain/MyProductApp.Domain.csproj
dotnet sln add src/MyProductApp.Application/MyProductApp.Application.csproj
dotnet sln add src/MyProductApp.Infrastructure/MyProductApp.Infrastructure.csproj
dotnet sln add src/MyProductApp.Api/MyProductApp.Api.csproj
```

添加项目引用以设置依赖关系：

```bash
dotnet add src/MyProductApp.Application/MyProductApp.Application.csproj reference src/MyProductApp.Domain/MyProductApp.Domain.csproj
dotnet add src/MyProductApp.Infrastructure/MyProductApp.Infrastructure.csproj reference src/MyProductApp.Domain/MyProductApp.Domain.csproj
dotnet add src/MyProductApp.Infrastructure/MyProductApp.Infrastructure.csproj reference src/MyProductApp.Application/MyProductApp.Application.csproj
dotnet add src/MyProductApp.Api/MyProductApp.Api.csproj reference src/MyProductApp.Application/MyProductApp.Application.csproj
dotnet add src/MyProductApp.Api/MyProductApp.Api.csproj reference src/MyProductApp.Infrastructure/MyProductApp.Infrastructure.csproj
```

## 领域层

包含实体、值对象、领域接口（仓储契约）和领域事件。

Entities/Product.cs

```csharp
namespace MyProductApp.Domain.Entities;

public class Product
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string Name { get; private set; } = null!;
    public string? Description { get; private set; }
    public decimal Price { get; private set; }
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;

    protected Product() { }

    public Product(string name, decimal price, string? description = null)
    {
        if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException("Name is required.", nameof(name));
        if (price < 0) throw new ArgumentException("Price must be >= 0", nameof(price));
        Name = name;
        Price = price;
        Description = description;
    }

    public void UpdatePrice(decimal newPrice)
    {
        if (newPrice < 0) throw new ArgumentException("Price must be >= 0");
        Price = newPrice;
    }

    public void UpdateName(string name)
    {
        if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException("Name is required.");
        Name = name;
    }
}
```

Interfaces/ISomeRepository.cs

```csharp
namespace MyProductApp.Domain.Interfaces;

using MyProductApp.Domain.Entities;

public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(Product product, CancellationToken ct = default);
    Task UpdateAsync(Product product, CancellationToken ct = default);
    Task DeleteAsync(Product product, CancellationToken ct = default);
    Task<List<Product>> ListAsync(CancellationToken ct = default);
}
```

领域层不依赖于 EF 或外部库。

## 应用层 — CQRS + DTOs + 接口

Application 层包含 MediatR 请求类型、DTO、验证器（可选）以及处理程序使用的接口。

向 Application 项目添加包：

```bash
dotnet add src/MyProductApp.Application package MediatR --version 12.*
dotnet add src/MyProductApp.Application package FluentValidation --version 11.*
```

DTOs/ProductDto.cs

```csharp
namespace MyProductApp.Application.DTOs;

public class ProductDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = null!;
    public string? Description { get; init; }
    public decimal Price { get; init; }
}
```

Commands/CreateProductCommand.cs

```csharp
using MediatR;
using MyProductApp.Application.DTOs;

public record CreateProductCommand(string Name, decimal Price, string? Description) : IRequest<ProductDto>;
```

Handlers/CreateProductHandler.cs

```csharp
using MediatR;
using MyProductApp.Domain.Interfaces;
using MyProductApp.Domain.Entities;
using MyProductApp.Application.DTOs;

public class CreateProductHandler : IRequestHandler<CreateProductCommand, ProductDto>
{
    private readonly IProductRepository _repository;

    public CreateProductHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<ProductDto> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var product = new Product(request.Name, request.Price, request.Description);
        await _repository.AddAsync(product, cancellationToken);
        return new ProductDto {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            Price = product.Price
        };
    }
}
```

Queries/GetProductByIdQuery.cs

```csharp
using MediatR;
using MyProductApp.Application.DTOs;

public record GetProductByIdQuery(Guid Id) : IRequest<ProductDto?>;
```

Handlers/GetProductByIdHandler.cs

```csharp
using MediatR;
using MyProductApp.Domain.Interfaces;
using MyProductApp.Application.DTOs;

public class GetProductByIdHandler : IRequestHandler<GetProductByIdQuery, ProductDto?>
{
    private readonly IProductRepository _repository;

    public GetProductByIdHandler(IProductRepository repository)
        => _repository = repository;

    public async Task<ProductDto?> Handle(GetProductByIdQuery request, CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id, cancellationToken);
        if (product == null) return null;
        return new ProductDto {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            Price = product.Price
        };
    }
}
```

## 基础设施层 — EF Core 实现

将 EF Core 和 SQLite 提供程序添加到基础设施中：

```bash
dotnet add src/MyProductApp.Infrastructure package Microsoft.EntityFrameworkCore --version 8.* 
dotnet add src/MyProductApp.Infrastructure package Microsoft.EntityFrameworkCore.Sqlite --version 8.*
dotnet add src/MyProductApp.Infrastructure package Microsoft.EntityFrameworkCore.Design --version 8.*
```

Persistence/AppDbContext.cs

```csharp
using Microsoft.EntityFrameworkCore;
using MyProductApp.Domain.Entities;

namespace MyProductApp.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; } = null!;

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<Product>(b =>
        {
            b.HasKey(p => p.Id);
            b.Property(p => p.Name).IsRequired().HasMaxLength(200);
            b.Property(p => p.Description).HasMaxLength(2000);
            b.Property(p => p.Price).HasPrecision(18,2);
            b.Property(p => p.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
        });
    }
}
```

Repositories/ProductRepository.cs

```csharp
using Microsoft.EntityFrameworkCore;
using MyProductApp.Domain.Entities;
using MyProductApp.Domain.Interfaces;
using MyProductApp.Infrastructure.Persistence;

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db) => _db = db;

    public async Task AddAsync(Product product, CancellationToken ct = default)
    {
        await _db.Products.AddAsync(product, ct);
        await _db.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(Product product, CancellationToken ct = default)
    {
        _db.Products.Remove(product);
        await _db.SaveChangesAsync(ct);
    }

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _db.Products.FindAsync(new object[] { id }, cancellationToken: ct);

    public async Task<List<Product>> ListAsync(CancellationToken ct = default)
        => await _db.Products.AsNoTracking().ToListAsync(ct);

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        _db.Products.Update(product);
        await _db.SaveChangesAsync(ct);
    }
}
```

## API 层 — 控制器与 Program.cs

现在连接 API。向 MyProductApp.Api 添加所需的包：

```bash
dotnet add src/MyProductApp.Api package MediatR --version 12.*
dotnet add src/MyProductApp.Api package Microsoft.EntityFrameworkCore.Sqlite --version 8.*
dotnet add src/MyProductApp.Api package Swashbuckle.AspNetCore --version 6.*
```

Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using MyProductApp.Infrastructure.Persistence;
using MyProductApp.Domain.Interfaces;
using MyProductApp.Infrastructure;
using MediatR;
using System.Reflection;

// builder
var builder = WebApplication.CreateBuilder(args);
// DbContext (SQLite for example)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection") ?? "Data Source=products.db"));
// Repositories
builder.Services.AddScoped<IProductRepository, ProductRepository>();
// MediatR scanning
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Assembly.Load("MyProductApp.Application")));
// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddControllers();

var app = builder.Build();
// auto-migrate (optional for demos)
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=products.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

Controllers/ProductController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using MediatR;

[ApiController]
[Route("api/[controller]")]
public class ProductController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductCommand cmd)
    {
        var dto = await _mediator.Send(cmd);
        return CreatedAtAction(nameof(Get), new { id = dto.Id }, dto);
    }
    
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var dto = await _mediator.Send(new GetProductByIdQuery(id));
        if (dto == null) return NotFound();
        return Ok(dto);
    }
}
```

## 迁移与运行应用程序

从解决方案的根目录，添加 EF 工具并创建迁移（目标是 Infrastructure 项目）：

```bash
# ensure dotnet-ef tool installed
dotnet tool install --global dotnet-ef

# from MyProductApp/src/MyProductApp.Infrastructure:
cd src/MyProductApp.Infrastructure
dotnet ef migrations add InitialCreate --startup-project ../MyProductApp.Api --output-dir Migrations
dotnet ef database update --startup-project ../MyProductApp.Api
```

或者，由于 Program.cs 在启动期间调用 db.Database.Migrate() ，如果包含迁移，应用程序将自动创建数据库。

运行 API：

```bash
cd ../MyProductApp.Api
dotnet run
```

在 <https://localhost:{port}/swagger> 处打开 Swagger，尝试使用 POST /api/product 和 GET /api/product/{id}接口。

## 验证与横切关注点

- 在应用层中使用 FluentValidation 来验证命令。注册一个管道行为来运行验证。
- 使用 MediatR 管道行为进行日志记录、性能测量、异常处理和验证。

验证器示例：

```csharp
using FluentValidation;

public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
         RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
         RuleFor(x => x.Price).GreaterThanOrEqualTo(0);
    }
}
```

注册为管道行为：

```csharp
dotnet add src/MyProductApp.Application package MediatR.Extensions.Microsoft.DependencyInjection
dotnet add src/MyProductApp.Application package FluentValidation.DependencyInjectionExtensions
// in Program.cs (Api) when registering MediatR:
builder.Services.AddValidatorsFromAssembly(Assembly.Load("MyProductApp.Application"));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

ValidationBehavior 是一个小型 MediatR 管道，它在处理程序之前运行验证器。

## 单元测试处理器

通过模拟 IProductRepository （使用 Moq 或 NSubstitute）来创建 MyProductApp.Application.Tests 并独立测试处理器。由于处理器仅依赖于接口，因此它们很容易进行单元测试。

使用 xUnit + Moq 的示例：

```csharp
public class CreateProductHandlerTests
{
    [Fact]
    public async Task Handle_Should_Add_Product_And_Return_Dto()
    {
        var repoMock = new Mock<IProductRepository>();
        repoMock.Setup(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
                .Returns(Task.CompletedTask);
        var handler = new CreateProductHandler(repoMock.Object);
        var cmd = new CreateProductCommand("Test", 9.99m, "desc");
        var result = await handler.Handle(cmd, CancellationToken.None);
        Assert.Equal("Test", result.Name);
        repoMock.Verify(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

## 设计说明与最佳实践

- 保持领域纯净：领域内部不包含 EF Core 和框架类型。
- 应用层中使用 DTO：避免将 EF 实体传递给 API。
- 处理器保持精简：仅协调操作；业务逻辑存在于领域实体和领域服务中。
- 领域层中定义仓储接口：在基础设施层中实现。
- 通过管道行为处理横切关注点：验证、性能、日志记录、缓存。
- 从小处着手：并非所有内容都需要 CQRS。将其用于复杂的写入/读取差异；保持简单情况的简单性。
- 事务管理：在基础设施层中，如果多个存储库需要单个事务，请考虑使用 DbContext + IUnitOfWork 。
- 在单独的启动项目中进行迁移：我们使用 API 作为启动项目；如果你愿意，可以创建一个单独的控制台应用程序用于迁移命令。

今天就到这里。期待与你再次相见。
