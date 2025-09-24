# [C# gRPC 服务中的健康检查](https://medium.com/@bsvrj9320/health-checks-in-grpc-service-86bc20df9241)

对 gRPC 服务执行健康检查是确保其可用且正常运行的重要实践。gRPC 健康检查协议在许多 gRPC 实现中得到支持，包括.NET。

## 在 gRPC 服务中实现健康检查的步骤

### 1. 安装所需的 NuGet 包

在你的 gRPC 服务项目中，你需要安装 Grpc.AspNetCore.HealthChecks 包以启用 gRPC 健康检查支持。

在你的包管理器控制台中运行此命令：

```bash
Install-Package Grpc.AspNetCore.HealthChecks
```

### 2. 在 Program.cs 中配置健康检查

针对 .NET 6+ 环境（在 Program.cs 中使用最小托管模型）：

```csharp
var builder = WebApplication.CreateBuilder(args);

// 添加 gRPC 和健康检查服务
builder.Services.AddGrpc();
builder.Services.AddGrpcHealthChecks();

// 构建应用
var app = builder.Build();

// 映射 gRPC 服务和健康检查端点
app.MapGrpcService<YourGrpcService>();  // 你的 gRPC 服务
app.MapGrpcHealthChecksService();       // 健康检查端点

app.Run();
```

对于早期版本（使用 Startup.cs ）：

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddGrpc();
        services.AddGrpcHealthChecks();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseRouting();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapGrpcService<YourGrpcService>();
            endpoints.MapGrpcHealthChecksService();
        });
    }
}
```

### 3. 实现你的 gRPC 服务

在你的 gRPC 服务中，通常的实现方式如下：

```csharp
public class YourGrpcService : YourGrpcService.YourGrpcServiceBase
{
    public override Task<YourResponse> YourRpcMethod(YourRequest request, ServerCallContext context)
    {
        // 实现你的业务逻辑
        return Task.FromResult(new YourResponse { ... });
    }
}
```

### 4. 访问健康检查

设置健康检查服务后，你可以使用 gRPC 健康检查协议查询 gRPC 服务器的健康状态。客户端或监控系统可以通过请求 grpc.health.v1.Health 服务来访问此信息。

你可以使用 grpc-health-probe CLI 工具（由 gRPC 提供）来查询健康状态：

```bash
grpc-health-probe -addr=localhost:5000
```

默认情况下，所有服务将被标记为 SERVING ，除非另有特别设置。

### 5. 自定义健康检查状态（可选）

你还可以为不同的服务设置自定义健康状态。例如，要报告特定服务的健康状况，你可以在 Program.cs 或 Startup.cs 中手动控制健康状态，如下所示：

```csharp
var healthService = app.Services.GetRequiredService<Grpc.HealthCheck.HealthServiceImpl>();
healthService.SetStatus("YourGrpcService", HealthCheckResponse.Types.ServingStatus.Serving);
healthService.SetStatus("AnotherService", HealthCheckResponse.Types.ServingStatus.NotServing);
```

### 6. 进阶：与 ASP.NET Core 健康检查集成

你还可以通过为数据库、缓存等依赖项添加自定义检查，将 gRPC 健康检查与 ASP.NET Core 的健康检查系统集成。

```csharp
builder.Services.AddGrpcHealthChecks()
    .AddCheck("Database", new SqlServerHealthCheck("YourConnectionString"));
```

通过这种方式，gRPC 健康检查可以与 ASP.NET Core 的更广泛健康检查系统相结合，从而提供对系统健康状况的更全面视图。

## 总结

- 安装 NuGet 包（ Grpc.AspNetCore.HealthChecks ）。
- 在 Program.cs 或 Startup.cs 中配置健康检查。
- 使用 app.MapGrpcHealthChecksService() 映射 gRPC 健康检查端点。
- 通过类客户端的 grpc-health-probe 使用 gRPC 健康检查协议访问健康状态。

今天就到这里。期待与你再次相见。
