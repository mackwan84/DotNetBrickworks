# [如何在 .NET 中为高流量调优 Kestrel](https://medium.com/@hadiyolworld007/kestrel-tuning-for-high-traffic-net-apis-fd3c999b1a73)

> 线程池调整、连接队列和 GC 调优 — 我如何使用 Kestrel 扩展我的 .NET 后端以处理疯狂流量。

学习如何在 .NET 中为高流量调优 Kestrel：优化线程池、队列限制和垃圾回收，打造极速 API。

## 问题所在：你的 .NET API 一直很快 — 直到它不再快

Kestrel 是运行在 ASP.NET Core 背后的强大 Web 服务器。它快速、轻量级且可用于生产环境。但默认情况下，它并未针对高并发环境进行优化。

也许这听起来很熟悉：

- 在负载下，请求排队，响应时间飙升
- CPU 使用率低，但吞吐量却很差
- 日志很干净，但 API 在大规模使用时"感觉"很慢

> 问题不在于你的代码——而在于 Kestrel 的默认设置。

今天，我将向你展示如何像调校赛车一样调整 Kestrel，从而释放强大的性能。

## 为什么 Kestrel 调优很重要

Kestrel 已针对通用工作负载进行了优化 — 但一旦你的请求量从每分钟 1K 增长到 100K，你需要：

- 提高线程并行度
- 提高连接队列阈值
- 控制 .NET 的内存管理

让我们来挨个看看这些调整。

### 配置线程池以匹配你的负载

.NET 使用线程池来处理传入的请求。在高流量情况下，创建新线程可能需要太长时间。

✅ 解决方案：增加最小线程数

在 Program.cs 或 Startup.cs 中，添加：

```csharp
using System.Threading;
ThreadPool.SetMinThreads(workerThreads: 200, completionPortThreads: 200);
```

这减少了流量高峰时的线程饥饿问题，并改善了冷启动行为。

提示：使用 dotnet-counters 进行监控，实时观察线程池饱和度。

### 增加 Kestrel 的请求队列限制

默认情况下，Kestrel 允许的并发请求数和排队连接数量有限。

✅ 解决方案：提高连接和请求队列阈值

在你的 appsettings.json 中：

```json
{
  "Kestrel": {
    "Limits": {
      "MaxConcurrentConnections": 10000,
      "MaxConcurrentUpgradedConnections": 5000,
      "MaxRequestBodySize": 104857600
    },
    "EndpointDefaults": {
      "Protocols": "Http1AndHttp2"
    }
  }
}
```

你也可以通过编程方式进行配置：

```csharp
webBuilder.ConfigureKestrel(serverOptions =>
{
    serverOptions.Limits.MaxConcurrentConnections = 10000;
    serverOptions.Limits.MaxConcurrentUpgradedConnections = 5000;
});
```

提示：使用 Application Insights 或 Prometheus 导出器来监控队列长度和拒绝阈值。

### 驯服垃圾回收器（GC）

如果没有为工作负载进行配置，.NET 的 GC 可能会暂停你的应用程序。

✅ 解决方案：使用服务器 GC + 低延迟模式

在 csproj 或 runtimeconfig.json 中，设置：

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true
    }
  }
}
```

并且可选择在高吞吐量操作周围使用 LowLatency 模式：

```csharp
using (new LatencyModeScope(GCLatencyMode.LowLatency))
{
    // 追求低延迟的工作
}
```

提示：使用 dotnet-trace 监控 GC 统计信息，以便在请求激增期间发现集合操作。

### 使用 Socket Transport 降低延迟（可选）

如果你需要每一微秒的性能：

```csharp
webBuilder.UseSockets(); // 替换默认 Libuv 传输
```

这会降低原始套接字吞吐量的系统开销——但在切换前请进行负载测试。

### 使用 dotnet-counters 测量一切

使用内置的 dotnet-counters 工具来跟踪：

```bash
dotnet-counters monitor --process-id <your_pid>
```

关键指标：

- 线程池线程数 ThreadPool Thread Count
- 请求排队数 Requests Queued
- GC 暂停时间 GC Pause Duration
- 连接队列长度 Connection Queue Length

## 最终思考：在不崩溃的情况下扩展

通过这些调整，我们能够处理：

- 每分钟 12 万次请求
- 0 个丢弃的连接
- P95 延迟低于 200 毫秒

最重要的是，我们没有触及业务逻辑就做到了这一点。

Kestrel 开箱即用已经很快 — 但经过调优后，它变得更加强大。

接下来试试这个：

- 在 Kestrel 前面设置 NGINX 或 YARP 作为反向代理
- 使用 BenchmarkDotNet 进行受控性能分析
- 与 Redis 或内存缓存搭配使用以减少 GC 压力

今天就到这里。欢迎大家留言分享你的 API 调优技巧。让我们更快地构建更好的后端。
