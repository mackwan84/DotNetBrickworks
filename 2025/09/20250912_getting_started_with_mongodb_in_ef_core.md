# [在 EF Core 中使用 MongoDB](https://antondevtips.com/blog/getting-started-with-mongodb-in-ef-core)

Entity Framework Core (EF Core) 是 .NET 中流行的 ORM，通常与 SQL Server 或 PostgreSQL 等关系型数据库一起使用。然而，随着 MongoDB 等 NoSQL 数据库的日益普及，开发人员经常需要将这些技术集成到他们的应用程序中。EF Core 8.0 引入了对 MongoDB 的支持，使得在 .NET 项目中使用你喜爱的 ORM 处理面向文档的数据变得更加容易。

今天，我将向你展示如何在 EF Core 8.0 中开始使用 MongoDB。

## 在 EF Core 8.0 中开始使用 MongoDB

### 步骤 1：设置 MongoDB

我们将使用 docker-compose-yml 在 docker 容器中设置 MongoDB：

```yaml
services:
  mongodb:
    image: mongo:latest
    container_name: mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=admin
    volumes:
      - ./docker_data/mongodb:/data/db
    ports:
      - "27017:27017"
    restart: always
    networks:
      - docker-web

networks:
  docker-web:
    driver: bridge
```

### 步骤 2：添加 MongoDB 驱动并连接到数据库

要在 EF Core 8.0 中连接到 MongoDB，你需要将官方的 MongoDB 驱动程序包添加到你的项目中：

```bash
dotnet add package MongoDB.EntityFrameworkCore
```

接下来，你需要在 appsettings.json 中配置到 MongoDB 的连接字符串：

```json
{
  "ConnectionStrings": {
    "MongoDb": "mongodb://admin:admin@mongodb:27017"
  }
}
```

### 步骤 3：创建 EF Core DbContext

首先，让我们创建一个 Shipment 实体：

```csharp
public class Shipment
{
    public required ObjectId Id { get; set; }
    public required string Number { get; set; }
    public required string OrderId { get; set; }
    public required Address Address { get; set; }
    public required string Carrier { get; set; }
    public required string ReceiverEmail { get; set; }
    public required ShipmentStatus Status { get; set; }
    public required List<ShipmentItem> Items { get; set; } = [];
    public required DateTime CreatedAt { get; set; }
    public required DateTime? UpdatedAt { get; set; }
}
```

这里的 ObjectId 表示 MongoDb 集合中的文档标识符。

在使用 MongoDB 时，你可以创建一个熟悉的 EF Core DbContext，就像使用 SQL 数据库时一样：

```csharp
public class EfCoreMongoDbContext(DbContextOptions options) : DbContext(options)
{
    public DbSet<Shipment> Shipments { get; init; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Shipment>()
            .ToCollection("shipments")
            .Property(x => x.Status).HasConversion<string>();
    }
}
```

你需要将所有实体添加到模型构建器中，并指定来自 MongoDB 的集合名称。现在你可以像使用 SQL 表一样使用 DbSet。

请注意，你可以为实体使用 Conversion 。在上面的代码中，我们使用字符串转换来确保枚举以字符串形式存储在数据库中。

最后，我们需要注册我们的 DbContext 并为其指定 UseMongoDB ：

```csharp
var mongoConnectionString = configuration.GetConnectionString("MongoDb")!;

builder.Services.AddDbContext<EfCoreMongoDbContext>(x => x
    .EnableSensitiveDataLogging()
    .UseMongoDB(mongoConnectionString, "shipping")
);
```

MongoDB 是一种 NoSQL 数据库，它不像 SQL 那样有严格的模式定义，因此你不需要创建任何迁移。

现在我们已经准备好在 MongoDB 中执行我们的第一个查询。

## 使用 EF Core 向 MongoDB 写入数据

使用 EF Core 向 MongoDB 写入数据与向 SQL Server 或 PostgreSQL 写入数据没有区别。使用 EF Core 配合 MongoDB 的真正好处在于，你并不知道底层实际上是在使用 NoSQL 数据库。

这样做的好处是，如果你使用 EF Core，只需进行少量调整就可以从 SQL 数据库迁移到 MongoDB。或者可以立即开始使用 MongoDB，而无需学习新的 API。

> 但请注意，并非所有 MongoDB 功能都在 EF Core 中可用。对于某些高级功能，你可能需要直接使用 MongoDB.Driver。

以下是如何使用 EF Core 在 MongoDB 中创建、更新和删除文档的方法：

```csharp
// 创建
var shipment = request.MapToShipment(shipmentNumber);
context.Shipments.Add(shipment);
await context.SaveChangesAsync(cancellationToken);

// 更新
shipment.Status = ShipmentStatus.Delivered;
await context.SaveChangesAsync(cancellationToken);

// 删除
context.Shipments.Remove(shipment);
await context.SaveChangesAsync(cancellationToken);
```

分配 Id 时，可以使用 ObjectId 工厂方法：

```csharp
shipment.Id = ObjectId.GenerateNewId();
```

## 使用 EF Core 从 MongoDB 读取数据

正如你可以猜到的，你可以使用 EF Core 中熟悉的 LINQ 方法来从 MongoDB 读取数据。

以下是如何选择单条或多条记录：

```csharp
var shipment = await context.Shipments
    .Where(x => x.Number == request.ShipmentNumber)
    .FirstOrDefaultAsync(cancellationToken: cancellationToken);

var shipments = await context.Shipments
    .Where(x => x.Status == ShipmentStatus.Dispatched)
    .ToListAsync(cancellationToken: cancellationToken);
```

你还可以在 FirstOrDefaultAsync 或其他类似方法中提供筛选谓词。

以下是检查任何实体是否存在或获取实体计数的方法：

```csharp
var shipmentAlreadyExists = await context.Shipments
    .Where(s => s.OrderId == request.OrderId)
    .AnyAsync(cancellationToken);
    
var count = await context.Shipments
    .CountAsync(x => x.Status == ShipmentStatus.Delivered);
```

## 在 Entity Framework 8.0 中使用 MongoDB 的限制

EF 8.0 中不支持以下 MongoDB 功能：

1. 选择投影：选择投影在 LINQ 查询中使用 Select() 方法来更改所创建对象的结构。EF 8 不支持此类投影。
2. 标量聚合：EF 8 仅支持以下标量聚合操作：
    - Count(), CountAsync()
    - LongCount(), LongCountAsync()
    - Any()、AnyAsync() 带或不带谓词
3. 事务：EF 8 不支持 MongoDB 的 Entity Framework Core 事务模型。

如果你需要无限制地使用这些功能中的任何一个 - 请直接使用 MongoDB.Driver 。

MongoDB 是一个 NoSQL 数据库，它不支持以下 EF Core 功能：

- 迁移
- 外键和备用键
- 拆分表和时态表
- 空间数据

有关更多信息，你可以阅读官方 MongoDB 文档。

## 总结

EF Core 8 允许通过熟悉的 Entity Framework 类、接口和方法与 MongoDB 进行无缝集成。你可以使用 DbContext 来配置 MongoDB 集合。可以使用你喜欢的 LINQ 方法执行读取操作，并使用熟悉的方法执行写入操作。

当然，在 Entity Framework 的后续版本中，将会为 MongoDB 添加越来越多的功能支持。

今天就到这里。期待与你再次相见。
