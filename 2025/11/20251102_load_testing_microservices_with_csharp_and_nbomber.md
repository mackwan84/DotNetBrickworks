# [使用 C# 和 NBomber 进行微服务负载测试](https://antondevtips.com/blog/load-testing-microservices-with-csharp-and-nbomber)

我已经从事微服务工作超过 10 年了。

没有这三样东西，你无法构建一个高质量的微服务系统：

- 可观测性
- 集成测试
- 负载测试

许多团队会跳过负载测试。他们认为这太耗时，或者负载测试并不重要。

这是一个错误。没有负载测试，问题通常会在生产环境中显现，那时修复已经为时已晚且成本高昂。

在微服务中，你的系统连接到许多部分：

- 数据库
- 分布式缓存
- 消息队列
- 外部 API
- 其他服务

你如何决定你的服务应该使用 MS SQL Server、PostgreSQL 还是 MongoDB？服务应该通过 REST API（同步通信）进行通信，还是使用消息队列（异步）？

负载测试将给你明确的答案。

它们显示了你的系统在变慢或崩溃前能够处理多少流量。它们揭示了系统中的缓慢部分，并帮助比较不同的设计选择。

它们显示真实的延迟数字，帮助你在压力下检查超时、重试和错误处理。

它们还为容量规划提供数据，让你知道何时需要扩展。当你定期运行这些测试时，你可以及早发现性能回归问题。

它们还帮助你在相同条件下比较不同的设置，例如 MS SQL Server 与 PostgreSQL 与 MongoDB。你可以测试 REST API 调用与消息队列事件，看看哪种通信方法更适合你的情况。

我发现在 .NET 中进行负载测试的最佳解决方案之一是 NBomber。

今天我想向你展示如何使用 NBomber 轻松地为微服务创建负载测试。

本文的在本文中，我们将探讨：

- NBomber 入门指南
- 使用 NBomber 创建负载测试
- 并行运行负载测试
- 使用 JSON 文件配置 NBomber
- 使用 NBomber 创建报告

闲话少说，让我们开始吧！

## NBomber 入门指南

NBomber 是一个现代且灵活的负载测试框架，旨在测试任何系统，无论其协议或语义模型（拉取/推送）如何。

NBomber 允许您使用纯 C# 或 F# 代码定义负载测试场景。与其他工具相比，它的一个关键优势是能够使用您喜欢的语言和 IDE（带调试器）来建模真实的负载测试。随着您的负载测试代码库的增长，使用类型安全编程语言对于维护和发展您的测试变得越来越重要。

NBomber 在设计上是协议无关的。与许多工具不同，它不依赖于特定协议的外部包，这使其足够灵活，可以测试从 HTTP 和 WebSockets 到 gRPC、Kafka、NoSQL 数据库或自定义协议的任何内容。

此外，NBomber 提供了丰富的开源插件和扩展生态系统 - 涵盖协议、实时报告、监控等。这使其非常适合在团队和组织范围的测试计划中采用。

NBomber 提供以下功能：

- 协议无关 – 您可以测试 HTTP、gRPC、WebSockets、数据库、消息代理或任何可以从 .NET 调用的内容。
- 云无关 – 您可以在任何云上运行和复制您的测试。无需依赖特定的云 UI，只需构建您的应用程序，部署并获取结果。
- 对开发者友好 – 您可以在 IDE 中使用 C#/F# 编写测试，具有类型安全和完整的调试支持。
- HTML 报告 – 带有自定义指标的内置 HTML 报告
- 实时报告 - 将负载测试指标存储在许多不同的可观测性系统中（DataDog、InfluxDB、TimeScaleDB、Grafana、NBomber Studio）
- 插件支持 - 你可以添加自己的插件（Converter、WebSockets、AMQP、MQTT）和数据接收器
- 与 xUnit/NUnit 集成 - 你可以将负载测试作为 CI/CD 流水线的一部分运行（xUnit 示例）。
- Kubernetes/集群模式 - 在多个节点上运行分布式测试。

## 创建场景

要创建负载测试，你需要定义一个场景。

场景代表你需要测试的某些用户行为，例如：

- 用户登录
- 用户加载个人资料页面
- 用户创建订单

您可以使用 Scenario.Create 方法创建一个场景：

```csharp
var scenario = Scenario.Create("order scenario", async context =>
{
    await Login();
    await LoadProfile();     
    await CreateOrder();

    return Response.Ok();        
})
.WithLoadSimulations(
    Simulation.Inject(rate: 10,
        interval: TimeSpan.FromSeconds(1),
        during: TimeSpan.FromSeconds(30))
);
```

当你运行此场景时，NBomber 将测量整个场景的执行情况。执行结束后，NBomber 将打印场景的统计结果。

每个负载测试包含以下阶段：

- 初始化
- 预热
- 轰炸（测试）
- 清理

## 场景初始化

此方法应用于初始化场景及其所有依赖项。您可以使用它来准备目标系统、填充数据库，或读取并应用场景的 JSON 配置。

```csharp
var scenario = Scenario.Create("order scenario", async context =>
{
    await Login();
    await LoadProfile();     
    await CreateOrder();
    
    return Response.Ok();
})
.WithInit(async context =>
{
    await SeedDatabase();
})
.WithLoadSimulations(
    Simulation.Inject(rate: 10,
        interval: TimeSpan.FromSeconds(1),
        during: TimeSpan.FromSeconds(30))
);
```

## 场景清理

此方法应在测试完成后用于清理场景的资源。

```csharp
var scenario = Scenario.Create("order scenario", async context =>
{
    await Login();
    await LoadProfile();     
    await CreateOrder();
    
    return Response.Ok();
})
.WithClean(async context =>
{
    await CleanDatabase();
});
```

让我们探讨如何为真实的微服务系统编写负载测试。

## 使用 NBomber 创建负载测试

我构建了一个包含三个服务的微服务应用程序：

- UserService — 负责用户身份验证和 JWT 签发
- ShipmentService — 用于创建货运
- StockService — 用于检查和更新库存

所有服务都部署在 API 网关（YARP）后面，并将数据存储在 PostgreSQL 中。

以下是创建货运时各服务之间的交互方式：

![Seq](https://antondevtips.com/media/code_screenshots/architecture/microservices-load-tests-nbomber/img_1.png)

"创建货运"用例具有以下流程：

1. ShipmentService 向 StockService 发送 REST HTTP 请求，检查是否有足够的产品来创建发货
2. 如果库存充足 — 在数据库中创建发货记录
3. 向消息队列发送 "ShipmentCreatedEvent"（发货创建事件）
4. StockService — 订阅该事件并更新库存

## 创建第一个负载场景

让我们用 NBomber 创建第一个场景，它包含 2 个步骤：

1. 用户登录
2. 用户加载他的个人资料

使用 NBomber，我们可以测试当许多用户同时登录其个人资料时 UserService 的性能表现。

按照以下简单步骤创建您的负载测试：

步骤 1：

创建一个 .NET 控制台项目。

步骤 2：

添加以下 NuGet 包：

```bash
dotnet add package NBomber
dotnet add package NBomber.Http
```

为了处理 HTTP，NBomber 为原生 .NET HttpClient 提供了 NBomber.Http 插件。该插件提供了有用的扩展，简化了创建和发送请求、接收响应以及跟踪数据传输和状态码的过程。

步骤 3：

使用 NBomber 创建测试场景。

你可以选择 2 个选项：

1. 你可以将所有需要的步骤作为一个整体运行，NBomber 将测量整个流程的性能：

```csharp
var httpClient = Http.CreateDefaultClient();

var scenario = Scenario.Create("user_login_and_get_profile_scenario", async context =>
{
    await Login();
    await GetProfile();
})
.WithoutWarmUp()
.WithLoadSimulations(
    Simulation.Inject(rate: 10,
        interval: TimeSpan.FromSeconds(1),
        during: TimeSpan.FromSeconds(30))
);
```

2. 或者你可以将每个方法移到单独的步骤中，NBomber 将分别测量它们：

```csharp
var httpClient = Http.CreateDefaultClient();

var scenario = Scenario.Create("user_login_and_get_profile_scenario",
    async context =>
{
    var step1 = await Step.Run("login", context, () => LoginStep(httpClient));

    var token = step1.Payload.Value;
    
    var step2 = await Step.Run("get_profile", context, () => GetProfileStep(httpClient, token));

    return step2;
})
.WithoutWarmUp()
.WithLoadSimulations(
    Simulation.Inject(rate: 10,
        interval: TimeSpan.FromSeconds(1),
        during: TimeSpan.FromSeconds(30))
);
```

以下是您如何使用 NBomber 发送 HTTP 请求：

```csharp
var loginRequest = Http.CreateRequest("POST", "http://localhost:5000/api/users/login")
    .WithJsonBody(new { Email = "admin@test.com", Password = "Test1234!"});

    // Send HTTP request with NBomber
    var loginResponse = await Http.Send(httpClient, loginRequest);
```

以下是每个步骤的完整实现：

```csharp

private async Task<Response<string>> LoginStep(
    HttpClient httpClient,
    IScenarioContext context)
{
    // Create HTTP request
    var loginRequest = Http.CreateRequest("POST", "http://localhost:5000/api/users/login")
        .WithJsonBody(new { Email = "admin@test.com", Password = "Test1234!"});

    var loginResponse = await Http.Send(httpClient, loginRequest);

    var responseBody = await loginResponse.Payload.Value.Content.ReadFromJsonAsync<LoginUserResponse>();

    return responseBody is null
        ? Response.Fail<string>(message: "No JWT token available")
        : Response.Ok(payload: responseBody.Token);
}

private async Task<Response<string>> GetProfileStep(
    HttpClient httpClient,
    IScenarioContext context)
{
    var profileRequest = Http.CreateRequest("GET", "http://localhost:5000/api/users/profile")
        .WithHeader("Accept", "application/json")
        .WithHeader("Authorization", $"Bearer {token}");

    var profileResponse = await Http.Send(httpClient, profileRequest);

    var responseBody = await profileResponse.Payload.Value.Content.ReadFromJsonAsync<UserResponse>();
    
    return responseBody is null
        ? Response.Fail<string>(message: "User profile not found")
        : Response.Ok<string>();
}
```

每个步骤和场景应返回 Response.Ok 或 Response.Fail 。

请注意， LoginStep 步骤返回在 GetProfileStep 中使用的 JWT 令牌。

您还可以使用 ScenarioContext 在步骤之间共享数据，它代表当前运行的场景的执行上下文。您可以使用 ScenarioContext 在步骤之间共享数据。

它提供了记录特定事件、获取有关测试、线程 ID、场景副本/实例编号等信息的功能。此外，它还提供了手动停止所有或特定场景的选项。

步骤 4：

在 Release 模式下运行场景：

```csharp
NBomberRunner
    .RegisterScenarios(scenario)
    .Run(args);
```

我们正在运行一个持续 30 秒的负载测试，测试结束后，我们将在控制台中获得以下结果：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/microservices-load-tests-nbomber/img_2.png)

该 30 秒测试以每秒 20 次请求的速度运行 user_login_and_get_profile_scenario ，共完成 600 次请求且零失败。

你可以看到一个登录请求耗时超过 1 秒——这是第一个请求，因为我们在创建场景时指定了 WithoutWarmUp 。

## 配置负载模拟

NBomber 为您提供了多种控制测试过程中如何添加虚拟用户的方法。这被称为负载模拟。

你可以配置：

- 开放模型：新虚拟用户以固定速率到达，无论已有多少用户正在运行。这适用于测试面临稳定请求流的系统。
- 封闭模型：活跃虚拟用户数量保持恒定。新用户只有在其他用户完成后才会启动。这对于测试具有可预测并发性的完整用户旅程非常有用。

您也可以在同一测试中结合这两种模型来模拟不同的流量模式。

对于大多数端到端用户流程，封闭模型是一个很好的起点。如果您不确定，请从这里开始并根据结果进行调整。

无论您选择哪种模型，都应逐步增加负载。这样可以避免系统立即过载，并帮助您发现性能开始下降的时间点。

## 在负载测试中使用 HttpClient 的最佳实践

在测试基于 HTTP 的服务时，请遵循以下最佳实践：

- 重用你的 HttpClient ——每个场景创建一个 HttpClient，而不是每个请求。这样可以避免不必要的 TCP 连接并减少开销。
- 每个用户的 HttpClient （如果需要）——如果每个虚拟用户需要自己的 cookie 或会话状态，请使用 context.ScenarioInstanceData 将专用的 HttpClient 附加到场景实例。
- 不要过于频繁地释放资源——频繁释放会导致套接字耗尽并使结果失真。在负载测试期间最好不要释放 HttpClient 。

这些实践将使您的负载测试保持可靠，并确保您是在测试服务的极限，而不是遇到测试代码中的问题。

## 创建第二个负载场景

让我们创建第二个场景，它包含 2 个步骤：

- 用户登录
- 用户创建了一个货件

```csharp
var httpClient = Http.CreateDefaultClient();

var scenario = Scenario.Create("user_login_and_create_shipment_scenario",
    async context =>
{
    var step1 = await Step.Run("login", context, () => LoginStep(httpClient));
    
    var token = step1.Payload.Value;

    var step2 = await Step.Run("create_shipment", context, () => CreateShipmentStep(httpClient, token));

    return step2;
})
.WithLoadSimulations(
    // ramp up the injection rate from 0 to 50 copies
    // injection interval: 5 seconds
    // duration: 30 seconds (it executes from [00:00:00] to [00:00:30])
    Simulation.RampingInject(rate: 50,
        interval: TimeSpan.FromSeconds(5),
        during: TimeSpan.FromSeconds(30)),

    // keeps injecting 50 copies per 1 second
    // injection interval: 5 seconds
    // duration: 30 seconds (it executes from [00:00:30] to [00:01:00])
    Simulation.Inject(rate: 50,
        interval: TimeSpan.FromSeconds(5),
        during: TimeSpan.FromSeconds(30)),

    // ramp down the injection rate from 50 to 0 copies
    // injection interval: 5 seconds
    // duration: 30 seconds (it executes from [00:01:00] to [00:01:30])
    Simulation.RampingInject(rate: 0,
        interval: TimeSpan.FromSeconds(5),
        during: TimeSpan.FromSeconds(30))
);
```

CreateShipmentStep 实现如下：

```csharp
private async Task<Response<string>> CreateShipmentStep(
    HttpClient httpClient,
    IScenarioContext context)
{
    var request = GetCreateShipmentRequest();

    // Send HTTP request with NBomber
    var createShipmentRequest = Http.CreateRequest("POST", "http://localhost:5000/api/shipments")
        .WithHeader("Content-Type", "application/json")
        .WithHeader("Authorization", $"Bearer {token}")
        .WithJsonBody(request);

    await Http.Send(httpClient, createShipmentRequest);

    return Response.Ok(payload: orderId);
}
```

让我们运行测试：

![测试结果](https://antondevtips.com/media/code_screenshots/architecture/microservices-load-tests-nbomber/img_3.png)

此测试验证我们的完整流程在负载下是否正常工作：

1. 用户登录
2. 用户发起创建货件
3. ShipmentService 向 StockService 发送 REST HTTP 请求，检查是否有足够的产品来创建发货
4. 如果库存充足 — 在数据库中创建发货记录
5. 向消息队列发送"ShipmentCreatedEvent"
6. StockService — 订阅该事件并更新库存

## 并行运行负载测试

在实际系统中，流量很少是单一类型的请求。有些用户可能在浏览个人资料，而另一些用户可能在创建订单，这两种情况同时发生。

NBomber 允许您通过并行运行多个场景来对此进行建模。例如，如果您的网站通常有 100 个并发用户，您可能会模拟：

- 70 位用户访问首页
- 30 个用户创建订单

您可以通过创建两个场景来实现这一点，每个场景都有其自己的吞吐量或虚拟用户数量，然后将它们注册在一起运行。这样，您的负载测试就能反映真实的流量模式，而不是单一的工作负载。

![](https://antondevtips.com/media/code_screenshots/architecture/microservices-load-tests-nbomber/img_4.jpg)

我们可以将之前的两个场景结合起来，并行地同时运行它们：

```csharp
var scenario1 = new UserLoginAndProfileScenario().CreateScenario();
var scenario2 = new UserLoginAndCreateShipmentScenario().CreateScenario();

NBomberRunner
    .RegisterScenarios(scenario1, scenario2)
    .Run(args);
```

```csharp
public class UserLoginAndProfileScenario
{
    public ScenarioProps CreateScenario()
    {
        var httpClient = Http.CreateDefaultClient();

        var scenario = Scenario.Create("user_login_and_get_profile_scenario",
            async context =>
        {
            // Step 1: LoginStep
            // Step 2: GetProfileStep
            // Code remains the same from the previous examples
        })
        .WithoutWarmUp()
        .WithLoadSimulations(
            Simulation.Inject(rate: 20,
                interval: TimeSpan.FromSeconds(1),
                during: TimeSpan.FromSeconds(30))
        );

        return scenario;
    }
}

public class UserLoginAndCreateShipmentScenario
{
    public ScenarioProps CreateScenario()
    {
        var httpClient = Http.CreateDefaultClient();

        var scenario = Scenario.Create("user_login_and_create_shipment_scenario",
            async context =>
        {
            // Step 1: LoginStep
            // Step 2: CreateShipmentStep
            // Code remains the same from the previous examples
        })
        .WithLoadSimulations(
            Simulation.RampingInject(rate: 50,
                interval: TimeSpan.FromSeconds(5),
                during: TimeSpan.FromSeconds(30)),

            Simulation.Inject(rate: 50,
                interval: TimeSpan.FromSeconds(5),
                during: TimeSpan.FromSeconds(30)),

            Simulation.RampingInject(rate: 0,
                interval: TimeSpan.FromSeconds(5),
                during: TimeSpan.FromSeconds(30))
        );

        return scenario;
    }
}
```

## 使用 JSON 文件配置 NBomber

NBomber 支持通过 JSON 文件进行配置，使得以下操作变得简单：

- 在不同环境下运行相同的测试
- 无需更改代码即可调整负载模式
- 将版本化的测试配置文件保存在源代码控制中

NBomber 支持两种 JSON 配置类型：

- JSON 配置 - 覆盖场景执行设置，如 LoadSimulations 、 WarmUpDuration 和 TargetScenarios 。
- JSON 基础设施配置 - 覆盖基础设施设置，如日志记录器、报告接收器和工作线程插件。

你可以通过三种方式加载 JSON：

```csharp
// From JSON file
NBomberRunner
    .RegisterScenarios(scenario)    
    .LoadConfig("config.json")
    .Run(args);

// Providing JSON directly
NBomberRunner
    .RegisterScenarios(scenario)    
    .LoadConfig("{ PLAIN JSON }")
    .Run(args);

// Providing from an URL
NBomberRunner
    .RegisterScenarios(scenario)    
    .LoadConfig("https://my-test-host.com/config.json")
    .Run(args);
```

NBomber 还支持用于加载 JSON 配置的 CLI 参数。

```bash
dotnet MyLoadTest.dll --config=config.json
```

## 配置负载模拟

你可以在 JSON 中完全定义负载模拟，包括多个阶段：

```json
{
    "LoadSimulationsSettings": [              
        { "RampingInject": [50, "00:00:01", "00:00:30"] },
        { "Inject": [50, "00:00:01", "00:01:00"] },
        { "RampingInject": [0, "00:00:01", "00:00:30"] }
    ]
}
```

此示例将请求速率提升至 50 RPS，保持稳定，然后逐渐降低。

## 配置阈值

您可以在 JSON 中设置阈值配置，这样当性能低于您的限制时，测试会自动失败：

```json
{
    "ThresholdSettings": [
        { "OkRequest": "RPS >= 30" },
        { "OkRequest": "Percent > 90" }
    ]
}
```

测试结束后，也可以在代码中检查阈值：

```csharp
var scenario = new UserLoginAndCreateShipmentScenario().CreateScenario();

var result = NBomberRunner
    .RegisterScenarios(scenario)
    .Run(args);

// Get the final stats to check against our thresholds.
var stats = result.ScenarioStats.Get("user_login_and_create_shipment_scenario");

Assert.True(stats.Fail.Request.Percent <= 5);
```

## 自定义指标

NBomber 会跟踪内置指标（CPU、RAM、数据传输），但你也可以为业务或技术 KPI 创建自定义指标。

两种常见类型是：

- 计数器 – 跟踪运行总数（例如，成功登录的总次数）。
- 仪表盘 – 跟踪最新值（例如，当前内存使用情况）。

以下是在代码中定义自定义指标的方法：

```csharp
// define custom metrics
var counter = Metric.CreateCounter("my-counter", unitOfMeasure: "MB");
var gauge = Metric.CreateGauge("my-gauge", unitOfMeasure: "KB");

var scenario = Scenario.Create("scenario", async context =>
{
    await Task.Delay(500);

    counter.Add(1); // tracks a value that may increase or decrease over time
    gauge.Set(6.5); // set the current value of the metric

    return Response.Ok();
})
.WithInit(ctx =>
{
    // register custom metrics
    ctx.RegisterMetric(counter);
    ctx.RegisterMetric(gauge);

    return Task.CompletedTask;
});

var stats = NBomberRunner
    .RegisterScenarios(scenario)
    .Run(args);

// We can retrieve the final metric values for further assertions
var counterValue = stats.Metrics.Counters.Find("my-counter").Value;
var gaugeValue = stats.Metrics.Gauges.Find("my-gauge").Value;
```

您还可以在自定义指标上设置阈值：

```csharp
// define custom metrics
var counter = Metric.CreateCounter("my-counter", unitOfMeasure: "MB");
var gauge = Metric.CreateGauge("my-gauge", unitOfMeasure: "KB");

var scenario = Scenario.Create("scenario", async context =>
{
    ...
})
.WithInit(ctx =>
{
    // register custom metrics
    ctx.RegisterMetric(counter);
    ctx.RegisterMetric(gauge);

    return Task.CompletedTask;
})
.WithThresholds(
    Threshold.Create(metric => metric.Counters.Get("my-counter").Value < 1000),
    Threshold.Create(metric => metric.Gauges.Get("my-gauge").Value >= 6.5)
);
```

## 使用 NBomber 创建报告

您可以使用 NBomber 生成报告。这些报告对于理解您的应用程序在负载下的表现至关重要。

这是一个您可以与之交互的 HTML 报告示例。

报告提供了对各种性能指标的详细洞察，帮助您识别瓶颈、优化性能并确保流畅的用户体验。

NBomber 可以生成以下格式的报告：

- HTML 报告
- MD 报告
- CSV 报告
- TXT 报告

您可以使用 WithReportFormats 方法配置报告格式：

```csharp
NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFormats(
        ReportFormat.Csv, ReportFormat.Html, 
        ReportFormat.Md, ReportFormat.Txt
    )
    .Run(args);
```

您也可以配置报告文件夹和文件名：

```csharp
NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFolder("reports")
    .WithReportFileName("shipment-reports")
    .Run(args);
```

NBomber 还允许您检索生成报告的内容。当您需要将它们上传到自定义目标（远程存储）时，这非常有用：

```csharp
var result = NBomberRunner
    .RegisterScenarios(scenario)
    .Run(args);

var htmlReport = result.ReportFiles.First(x => x.ReportFormat == ReportFormat.Html);

var filePath = htmlReport.FilePath;  // file path
var html = htmlReport.ReportContent; // string HTML content

// You can upload the content to a remote storage
```

当我们并行运行两个场景时，HTML 报告如下所示：

![HTML 报告](https://antondevtips.com/media/code_screenshots/architecture/microservices-load-tests-nbomber/img_5.png)

在"场景"选项卡中，您可以查看在测试会话期间运行的所有场景。

## 结论

NBomber 是一个强大且灵活的 .NET 负载测试工具。它允许您创建真实的用户场景，并行运行这些场景，并在每一步测量性能。

您可以在同一个测试中模拟 HTTP、gRPC、WebSockets、数据库以及像 RabbitMQ 这样的消息队列。

其 JSON 配置、运行时阈值和自定义指标使测试调整变得简单。内置报告显示延迟百分位数、错误率和系统指标，让您能够快速发现问题。

使用 NBomber，您可以验证系统限制、比较技术选择，并在问题到达生产环境之前找到瓶颈。

NBomber 最大的优势是您可以直接用 C#或 F#编写负载测试。这意味着您可以在测试中重用现有的应用程序代码、辅助程序和库。您不需要学习 JavaScript 或自定义脚本来创建测试。

NBomber 供个人使用免费。在组织中使用 NBomber 需要付费许可证，在此处了解更多信息。

NBomber 的定价非常实惠，因为其许可证覆盖整个组织。单个许可证可以在所有团队之间共享，因此无需管理单独的开发者席位——一个许可证即可适用于整个公司。

立即开始使用 NBomber 构建您的负载测试，在问题进入生产环境之前发现它们：

- 我强烈建议从 Hello World 教程开始。
- 那么请阅读这篇关于微服务负载测试的文章。它将为您提供如何通过隔离式和端到端(E2E)负载测试来覆盖您系统的基础知识。
- 之后，您就可以探索他们的演示示例集合了

今天就到这里。希望这些内容对你有帮助。
