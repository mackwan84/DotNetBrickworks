# [如何使用 Seq 实现微服务的结构化日志和分布式追踪](https://antondevtips.com/blog/how-to-implement-structured-logging-and-distributed-tracing-for-microservices-with-seq)

在现代微服务架构中，可观测性对于理解服务的交互和行为至关重要。实施结构化日志和分布式追踪有助于诊断问题并监控系统性能。在这篇博客文章中，我们将探讨如何使用 Seq 来实现这些目的。

Seq 是一个集中式日志服务器，允许你收集和分析结构化日志事件。它提供了强大的查询功能和一个用户友好的界面，用于可视化日志和追踪。从 2024.1 版本开始，Seq 完全支持分布式追踪，包括 OpenTelemetry 追踪摄入、追踪索引和分层追踪可视化。

之前我一直在使用 Jaeger 最受欢迎的分布式追踪服务之一。**但现在，Seq 允许在一个地方查看日志和追踪，因此我觉得它比分别使用两个独立的服务来处理日志和分布式追踪更加方便。**

Seq 提供免费和付费许可证，正如他们在 GitHub 上所声明的那样，只要你符合“个人”要求，就可以在开发和生产环境中使用 Seq。

现在让我们深入了解结构化日志、分布式追踪和 OpenTelemetry。

## 什么是结构化日志？

- **增强查询**：你可以根据特定事件参数过滤和搜索日志。
- **更好的分析**：日志可以更有效地聚合和可视化。
- **机器可读**：像 Seq 这样的工具可以解析并以有意义的方式显示日志。

## 什么是分布式追踪？

分布式追踪允许在请求穿越分布式系统中的各个组件或微服务时进行监控和故障排除。它有助于可视化请求的流程，并识别瓶颈或故障。

### 分布式追踪的关键概念

1. **Spans 跨度**：跟踪中的单个工作单元。通常，表示对一个外部系统（数据库、缓存等）的一次请求。
2. **Traces 跟踪**：表示跨多个服务的请求的一组跨度。
3. **Context Propagation 上下文传播**：跨服务边界传递跟踪上下文以关联跨度。

## 什么是 OpenTelemetry？

OpenTelemetry 是一个开源的可观测性框架，提供 API、库和工具，用于仪器化、生成、收集和导出遥测数据（指标、日志和跟踪）以进行分析。

OpenTelemetry 的主要特点：

- 支持多种编程语言。
- 供应商中立：适用于不同的后端（例如，Seq、Jaeger、Prometheus）。
- 为各种框架提供开箱即用的测量化工具。

## 如何使用 Seq 实现结构化日志和分布式追踪？

要使用 Seq 实现结构化日志和分布式追踪，我们需要遵循以下步骤：

1. 在你的机器上安装 Seq 服务器，或者启动 Seq Docker 容器。
2. 配置你的微服务以将结构化日志事件发送到 Seq。
3. 使用 OpenTelemetry 为你的微服务进行仪器化，以生成和传播跟踪。
4. 配置 Seq 以摄取和索引跟踪。
5. 使用 Seq 用户界面来可视化和分析日志和跟踪。

接下来让我们深入探讨每个步骤的细节。

今天我们将构建两个微服务：**ShippingService** 和 **OrderTrackingService**。

**ShippingService** 将负责为已购买的产品创建和更新发货信息。

**OrderTrackingService** 将负责跟踪货物状态，并通过电子邮件通知用户货物更新的情况。

我们将构建这些服务的简化版本，以便向你展示如何实现结构化日志记录和分布式追踪。稍后我们将探索 Seq，以可视化所有这些数据。

我们将使用以下技术来创建 ShippingService 和 OrderTrackingService：

- ASP.NET Core、Minimal API（用于组织代码的垂直切片架构）
- MongoDB（用于数据存储）
- MediatR（用于 CQRS 模式）
- MassTransit（用于基于 RabbitMQ 的事件驱动架构）
- Refit（用于服务之间的同步通信）
- Redis（用于缓存）
- MailKit（用于发送电子邮件）
- OpenTelemetry（用于分布式追踪）
- Serilog（用于结构化日志记录）
- Docker 用于运行 MongoDB、RabbitMQ、Redis 和 Seq 容器

> 注意：在构建 Shipping 和 OrderTracking 微服务时，我使用了垂直切片架构；你可以构建自己的垂直切片，使用清洁架构或更简单的分层架构。

## 步骤 1：安装必要的 Docker 容器

首先，你需要使用 docker-compose-yml 安装 MondoDB、RabbitMQ、Redis 和 Seq 的 Docker 容器：

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

  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "15672:15672"
      - "5672:5672"
    restart: always
    volumes:
      - ./docker_data/rabbitmq:/var/lib/rabbitmq
    networks:
      - docker-web

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    restart: always
    volumes:
      - ./docker_data/redis:/data
    networks:
      - docker-web

  seq:
    image: datalust/seq:2024.3
    container_name: seq
    restart: always
    environment:
      - ACCEPT_EULA=Y
    volumes:
      - ./docker_data/seq:/data
    ports:
      - "5341:5341"
      - "8081:80"
    networks:
      - docker-web

networks:
  docker-web:
    driver: bridge
```

> 为了确保数据在 Docker 重启后仍然持久化，我们为所有 Docker 容器配置了卷。

## 步骤 2：实现运输和订单跟踪微服务

### ShippingService

ShippingService 实现了以下用例，可通过 webapi 公开访问：

- 创建发货
- 更新货物状态
- 按编号获取发货信息

1. 创建发货：将发货详情保存到 MongoDB，向 RabbitMQ 发布 ShipmentCreatedEvent ，并返回一个发货编号。
2. 更新货物状态：在 MongoDB 中更新货物的状态，并向 RabbitMQ 发布 ShipmentStatusUpdatedEvent 。
3. 根据编号获取运单：从 MongoDB 检索运单信息。

让我们来看看“创建发货”用例的实现：

```csharp
private sealed record CreateShipmentRequest(
    string OrderId,
    Address Address,
    string Carrier,
    string ReceiverEmail,
    List<ShipmentItem> Items);
    
internal sealed record CreateShipmentResponse(string ShipmentNumber);

app.MapPost("/api/shipments",
    async ([FromBody] CreateShipmentRequest request, IMediator mediator) =>
    {
        var command = new CreateShipmentCommand(
            request.OrderId,
            request.Address,
            request.Carrier,
            request.ReceiverEmail,
            request.Items);

        var response = await mediator.Send(command);
        if (response.IsError)
        {
            return Results.BadRequest(response.Errors);
        }

        return Results.Ok(response.Value);
    });
```

我使用 Minimal API 来创建 WebAPI 端点，并使用 MediatR 库实现 CQRS 模式。在这里，我创建了一个用于创建发货的命令，让我们探讨一下命令处理器的 Handle 方法：

```csharp
public async Task<ErrorOr<CreateShipmentResponse>> Handle(
    CreateShipmentCommand request,
    CancellationToken cancellationToken)
{
    var shipmentAlreadyExists = await context.Shipments
        .Find(s => s.OrderId == request.OrderId)
        .AnyAsync(cancellationToken);

    if (shipmentAlreadyExists)
    {
        logger.LogInformation("Shipment for order '{OrderId}' is already created", request.OrderId);
        return Error.Failure($"Shipment for order '{request.OrderId}' is already created");
    }

    var shipmentNumber = new Faker().Commerce.Ean8();
    var shipment = CreateShipment(request, shipmentNumber);

    await context.Shipments.InsertOneAsync(shipment, cancellationToken: cancellationToken);

    logger.LogInformation("Created shipment: {@Shipment}", shipment);

    var shipmentCreatedEvent = CreateShipmentCreatedEvent(shipment);
    await bus.Publish(shipmentCreatedEvent, cancellationToken);

    return new CreateShipmentResponse(shipment.Number);
}
```

在货物信息保存到数据库后，使用 MassTransit 向 RabbitMQ 发送一个 ShipmentCreatedEvent 。在处理 MongoDB 时，我喜欢创建一个 MongoDbContext 类，该类封装了所有 IMongoCollections ：

```csharp
public class MongoDbContext(IMongoClient mongoClient)
{
    private readonly IMongoDatabase _database = mongoClient.GetDatabase("shipping");

    public IMongoCollection<Shipment> Shipments => _database.GetCollection<Shipment>("shipments");
}
```

我发现这种方法很有用，因为我可以把所有的数据库和集合名称集中在一个地方。

对于请求-响应，我喜欢使用位置记录。对于实体 - 使用仅初始化属性且标记为必需的类。

对于我的应用逻辑类，比如 CommandHandlers，我喜欢使用主构造函数来避免样板代码。

我的这个用例的垂直切片如下所示：

![代码片段](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_1.png)

### OrderTrackingService

OrderTrackingService 实现了以下用例：

- 创建跟踪（内部）
- 更新跟踪状态（内部）
- 按编号获取跟踪信息（公开，可在 Web API 中使用）

1. 创建跟踪：将货件跟踪信息保存到 MongoDB 和 Redis。
2. 更新跟踪状态：从 RabbitMQ 消费运单状态更新，更新 MongoDB，刷新 Redis 缓存，并向用户发送电子邮件通知。
OrderTrackingService 消费来自 RabbitMQ 的 ShipmentCreatedEvent 和 ShipmentStatusUpdatedEvent 事件。为了提高应用程序的性能，跟踪信息被保存到 Redis 中，以便在客户端请求货运状态时进行快速数据读取。
3. 通过编号获取追踪信息：根据给定的编号返回被追踪货物的信息。

让我们来看看这个用例的实现：

```csharp
internal sealed record TrackingResponse(
    string Number,
    ShipmentStatus Status,
    ShipmentDetails ShipmentDetails,
    List<TrackingHistoryItem> HistoryItems);
    
app.MapGet("/api/tracking/{trackingNumber}",
    async ([FromRoute] string trackingNumber, IMediator mediator) =>
{
    var response = await mediator.Send(new GetTrackingByNumberQuery(trackingNumber));
    return response is not null ? Results.Ok(response) : Results.NotFound($"Tracking with number '{trackingNumber}' not found");
});
```

“接收货物更新”查询处理器首先在 Redis 中查找被跟踪的货物，然后在 MongoDB 中查找，如果仍未找到，则向 ShippingService 发送 HTTP 请求以检索货物。

让我们逐步探索查询处理器的 Handle 方法：

```csharp
public async Task<TrackingResponse?> Handle(GetTrackingByNumberQuery request, CancellationToken cancellationToken)
{
    // 首先在 Redis 中查找
    var tracking = await GetTrackingFromCacheAsync(request.TrackingNumber);
    if (tracking is not null)
    {
        return CreateTrackingResponse(tracking);
    }
    
    // 如果 Redis 中没有找到，则从 MongoDB 中查找
    tracking = await GetTrackingFromDbAsync(request, cancellationToken);
    if (tracking is not null)
    {
        await SaveTrackingInCacheAsync(tracking);
        return CreateTrackingResponse(tracking);
    }

    // ...
}
```

如果在 Redis 和 Mongo 中都未找到追踪信息，将发送 HTTP 请求到 ShippingService 以获取发货详情：

```csharp
logger.LogInformation("Tracking with number {TrackingNumber} not found in the database. Sending request to shipping-service", request.TrackingNumber);

var shipment = await shippingService.GetShipmentAsync(request.TrackingNumber);
if (shipment is null)
{
    logger.LogDebug("Shipment by tracking number {TrackingNumber} not found", request.TrackingNumber);
    return null;
}
```

如果找到发货信息 - 我们可以在 OrderTrackingService MongoDB 中创建跟踪并将其保存到 Redis：

```csharp
var command = new CreateTracking.CreateTrackingCommand(shipment);

var response = await mediator.Send(command, cancellationToken);
if (response.IsError)
{
    logger.LogDebug("Tracking with number {TrackingNumber} not found", request.TrackingNumber);
    return null;
}

tracking = response.Value;

await SaveTrackingInCacheAsync(tracking);

return CreateTrackingResponse(tracking);
```

OrderTrackingService 拥有订阅来自 ShippingService 事件的 RabbitMQ 消费者，让我们来看看使用 MassTransit 的实现方式：

```csharp
public async Task Consume(ConsumeContext<ShipmentCreatedEvent> context)
{
    var message = context.Message;

    logger.LogInformation("Received shipment created event: {@Event}", message);

    var shipment = CreateShipment(message);
    var command = new CreateTracking.CreateTrackingCommand(shipment);

    var response = await mediator.Send(command, context.CancellationToken);
    if (response.IsError)
    {
        logger.LogDebug("Shipment by tracking number {TrackingNumber} not found", message.Number);
        return;
    }

    await SaveTrackingInCacheAsync(response.Value);
}

public async Task Consume(ConsumeContext<ShipmentStatusUpdatedEvent> context)
{
    var message = context.Message;

    logger.LogInformation("Received shipment status updated event: {@Event}", message);

    var command = new UpdateTrackingStatus.UpdateTrackingCommand(message.ShipmentNumber, message.Status);

    var response = await mediator.Send(command);
    if (response.IsError)
    {
        logger.LogDebug("Shipment by tracking number {TrackingNumber} not found", message.ShipmentNumber);
        return;
    }

    await SaveTrackingInCacheAsync(response.Value);
}
```

这些消费者正在发送相应的中介命令。

现在你已经很好地理解了这些服务是如何协同工作的。在下一步中，我们将使用 Serilog 配置结构化日志，并将它们发送到 Seq。

### 步骤 3：配置微服务以将结构化日志事件发送到 Seq

要配置你的微服务以将结构化日志事件发送到 Seq，你需要：

1. 将 Serilog 日志包添加到两个微服务中。
2. 为每个服务配置 Serilog 日志设置
3. 配置 Serilog 将日志发送到 Seq 服务器
4. 编写必要的日志事件。

Serilog 是一个快速的日志库，它允许你记录结构化数据（键值对），而不是纯文本，从而使查询和分析日志变得更加容易。

我们将使用 Serilog 的以下方面：

- **结构化日志**：Serilog 允许你记录结构化数据（键值对），而不是纯文本，这使得查询和分析日志变得更加容易。
- **丰富数据上下文**：你可以通过添加上下文信息来丰富日志事件，例如用户 ID、请求 ID 以及其他相关元数据，从而更深入地了解应用程序行为。
- **灵活的输出目标**：Serilog 支持多种输出目标（sinks），包括控制台、文件、数据库和云日志平台，允许你将日志定向到最适合你需求的存储位置。

首先，我们需要添加以下 Nuget 包（ShippingService）：

```xml
<PackageReference Include="Serilog.AspNetCore" Version="8.0.1" />
<PackageReference Include="Serilog.Sinks.Console" Version="6.0.0" />
<PackageReference Include="Serilog.Sinks.Seq" Version="8.0.0" />
```

我们将使用 Serilog.Sinks.Seq 将日志事件发送到 Docker 容器中的 Seq。

现在，我们可以将 Serilog 日志配置添加到 appsettings.json ：

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Console",
      "Serilog.Sinks.Seq"
    ],
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft": "Information"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://localhost:5341"
        }
      }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ],
    "Properties": {
      "Application": "ShippingService"
    }
  }
}
```

在这里，我们配置日志输出到控制台（生产环境中不要使用）和 Seq。在本地运行时，我们将 Seq 的 URL 指向 <http://localhost:5341> 。当在 Docker 容器内运行服务时，需要使用 Docker 容器的名称而不是 localhost： <http://seq:5341> 。

请注意，我们指定了一个 "Application": "ShippingService" ，这在查看 Seq 中的日志和跟踪时，对于区分应用程序源非常重要。

完成 Serilog 的配置 - 将其添加到 WebApplicationBuilder ：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog((context, loggerConfig) => loggerConfig.ReadFrom.Configuration(context.Configuration));
```

这种方法的优点在于，你可以使用 Microsoft ILogger 接口进行日志记录，它在底层调用 Serilog。

现在让我们看看在 CreateShipmentHandler 中是如何进行结构化日志记录的：

```csharp
var shipmentAlreadyExists = await context.Shipments
    .Find(s => s.OrderId == request.OrderId)
    .AnyAsync(cancellationToken);

if (shipmentAlreadyExists)
{
    logger.LogInformation("Shipment for order '{OrderId}' is already created", request.OrderId);
    return Error.Failure($"Shipment for order '{request.OrderId}' is already created");
}

var shipmentNumber = new Faker().Commerce.Ean8();
var shipment = CreateShipment(request, shipmentNumber);

await context.Shipments.InsertOneAsync(shipment, cancellationToken: cancellationToken);

logger.LogInformation("Created shipment: {@Shipment}", shipment);

// ...
```

日志消息“订单‘{OrderId}’的发货单已创建”包含了 OrderId 作为一个结构化属性。而不是将 OrderId 直接嵌入日志消息中作为纯文本，它是作为命名参数传递的。这使得日志系统能够将 OrderId 捕获为一个独立的、可搜索的字段。

请千万不要这样进行日志记录，否则你最终会得到无法通过重要参数进行搜索的纯文本日志：

```csharp
logger.LogInformation($"Shipment for order '{request.OrderId}' is already created");
```

日志消息“已创建发货：{@Shipment}”使用@符号将发货对象序列化为结构化格式。这意味着发货对象的所有属性都作为单独的字段记录，保留了结构，使得分析更加容易。

另一个结构化日志的例子可能是：

```csharp
logger.LogInformation("Shipment for order '{OrderId}' is already created", request.OrderId);
logger.LogInformation("Updated state of shipment {ShipmentNumber} to {NewState}", request.ShipmentNumber, request.Status);
```

通过以这种结构化的方式实现日志记录，你将能够在 Seq 中搜索日志，例如，获取与特定 ShipmentNumber、State 或 OrderId 相关的所有事件。

让我们探讨一个示例，其中我们搜索更新状态为以下值之一的货物："处理中"、"已派发"、"在途中"：

现在我们已经部署了结构化日志，让我们将 OpenTelemetry 添加到我们的微服务中。

### 使用 OpenTelemetry 为微服务进行追踪 Instrumentation

要为你的微服务使用 OpenTelemetry 进行追踪，请按照以下步骤操作：

1. 将 OpenTelemetry 包添加到每个微服务项目中。
2. 配置 OpenTelemetry 跟踪器以生成和传播跟踪。
3. 设置必要的跟踪上下文传播。

首先，我们需要添加以下 Nuget 包（OrderTrackingService）：

```xml
<PackageReference Include="MongoDB.Driver.Core.Extensions.DiagnosticSources" Version="1.4.0"/>
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.8.1"/>
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.8.1"/>
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.8.1"/>
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.8.1"/>
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="1.8.1"/>
<PackageReference Include="OpenTelemetry.Instrumentation.StackExchangeRedis" Version="1.0.0-rc9.14"/>
```

要配置跟踪，你需要将 OpenTelemetry 添加到依赖注入（DI）中：

```csharp
services
    .AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("OrderTrackingService"))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRedisInstrumentation()
            .AddSource(MassTransit.Logging.DiagnosticHeaders.DefaultListenerName)
            .AddSource("MongoDB.Driver.Core.Extensions.DiagnosticSources")
            .AddSource("MailKit");

        tracing.AddOtlpExporter();
    });
```

你需要通过指定服务名称并添加适当的跟踪工具来配置资源：

- AddAspNetCoreInstrumentation - 添加 asp.net core 跟踪
- AddHttpClientInstrumentation - 在使用 HTTP 客户端发送请求时添加跟踪
- AddRedisInstrumentation - 添加来自 Redis 的跟踪
- AddSource("MassTransit") - 为 MassTransit 和 RabbitMQ 添加跟踪
- AddSource("MongoDB") - 为 MongoDB 添加跟踪
- AddSource("MailKit") - 为使用 MailKit 发送电子邮件添加跟踪（自定义实现）

最后，你需要添加 OpenTelemetry 导出器。

随着我们的跟踪准备就绪——让我们配置 Seq 以摄取这些跟踪。

### 步骤 5：配置 Seq 以摄取和索引跟踪

要配置 Seq 以接收和索引跟踪 - 配置 OpenTelemetry 导出器将跟踪发送到 Seq 服务器。你可以在代码中使用 tracing.AddOtlpExporter(); 方法进行配置，或者在 appsettings.json 中设置。我更倾向于后者：

```json
{
  "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:5341/ingest/otlp/v1/traces",
  "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf"
}
```

在 Docker 容器内运行服务时，需要使用 Docker 容器的名称而不是 localhost: <http://seq:5341> 。

在运行生产模式时，你需要创建一个 API 密钥。我喜欢创建两个独立的 API 密钥：一个用于日志记录，另一个用于跟踪。日志记录 API 密钥可以配置为接受特定日志级别的日志事件，如 Information 或 Debug ，根据你的需求。跟踪 API 密钥应在不指定任何特定日志级别的情况下创建，否则跟踪信息不会被 Seq 接收。

你可以在 appsettings.json 中指定 API 密钥（为简洁起见，省略了一些 Serilog 部分）：

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Seq"
    ],
    "MinimumLevel": { },
    "WriteTo": [
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "http://seq:5341",
          "apiKey": "abcde123"
        }
      }
    ]
  },
  "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:5341/ingest/otlp/v1/traces",
  "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
  "OTEL_EXPORTER_OTLP_HEADERS": "X-Seq-ApiKey=abcde12345"
}
```

现在，一切准备就绪，我们终于可以看看 Seq 如何可视化我们的结构化日志和 OpenTelemetry 跟踪了。

### 步骤 6：在 Seq 中可视化和分析日志和追踪

在 Seq 中可视化和分析日志和跟踪，请使用 Seq 用户界面：

1. 打开 Seq 用户界面。
2. 导航至“事件”部分。
3. 使用强大的查询功能来过滤和搜索日志和追踪。
4. 探索分层跟踪可视化，以理解请求的流程。

现在让我们探讨一些结构化日志和跟踪的示例。

1. "创建发货"用例：

![创建发货](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_3.png)

正如你在这张截图上所看到的，我们涉及了两个服务：ShippingService（蓝色轨迹）和 OrderTrackingService（紫色轨迹）。ShippingService 将其运单保存到数据库，并使用 MassTransit 向 RabbitMQ 发送一个 ShipmentCreatedEvent 。之后，OrderTrackingService 消费该事件，将运单跟踪信息保存到 MongoDB 和 Redis。

你可以查看此特定跟踪（创建运单的 WebAPI 请求）的所有日志，了解涉及了哪些组件、它们执行了哪些操作以及这些操作花费了多少时间。给定跟踪的每个操作称为一个跨度。在 Seq 中，你可以选择一个跨度以查看更多详细信息：

![创建发货](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_4.png)

![创建发货](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_5.png)

2. “更新发货状态”用例：

![更新发货状态](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_6.png)]

正如你在这张截图上看到的，这个操作几乎用了 3 秒钟。为什么会这么久，你可能会问？Seq 给出了答案——发送邮件几乎占用了所有时间。邮件是在后台的 ShipmentStatusUpdatedEvent 消费者中发送的，这是可以的，因为它不会减慢整个应用程序的速度，也不会让用户等待。

3. “根据编号获取跟踪信息”用例：

![根据编号获取跟踪信息](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_7.png)

此跟踪展示了如我之前描述的此用例的流程。OrderTrackingService 尝试在 Redis Cache 中查找跟踪信息，如果没有找到，则去 MongoDB 中查找，如果仍然没有找到，则向 ShippingService 发送 HTTP 请求以检索正在跟踪的货物。检索到货物后，OrderTrackingService 继续其工作。

4. 查找错误：

Seq 允许你快速找到日志中的错误：

![查找错误](https://antondevtips.com/media/code_screenshots/architecture/seq-logging-traces/img_aspnet_seq_8.png)

如你所见，此请求失败，因为在 ShippingService 中未找到跟踪的货物。

## 总结

Seq 允许你快速找到应用程序中的任何问题，如错误、缓慢或失败的请求。它提供了对复杂分布式系统中请求流的全面理解，使故障排除和应用程序调试变得更加容易。

Seq 的日志功能让你能够更快、更高效地分析日志数据。你可以选择日志以获取统计数据、洞察信息，并轻松地将所有统计数据以图表形式可视化。你还可以在 Seq 中配置警报，以便在出现错误、慢请求、重要日志消息或任何需要关注的事项时收到通知。

就是这样！按照这些步骤，你可以在你的微服务架构中使用 Seq 进行结构化日志记录和分布式追踪。

使用本手册，你可以将 Serilog 和 OpenTelemetry 导出器配置为将日志和跟踪发送到你选择的另一个工具，如果 Seq 不符合你的需求。你只需在 appsettings.json 中进行一些小调整，而无需触及代码。

今天就到这里。期待与你再次相见。
