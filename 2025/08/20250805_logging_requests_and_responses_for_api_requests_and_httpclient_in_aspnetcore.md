# [在 ASP.NET Core 中记录 API 和 HttpClient 的请求和响应日志](https://antondevtips.com/blog/logging-requests-and-responses-for-api-requests-and-httpclient-in-aspnetcore)

记录 HTTP 请求和响应有助于开发人员快速排查问题、监控性能和健康状况，以及审计其应用程序中的用户交互。ASP.NET Core 通过 HttpLogging 提供内置支持，你可以根据特定需求轻松配置和扩展它。

今天，我们将深入探讨：

- 如何在 ASP.NET Core 项目中启用和配置 HTTP 日志记录
- 日志记录选项和设置
- 自定义日志和使用特定于端点的配置
- 如何在日志中删改敏感数据
- 如何在使用 HttpClient 发送请求时记录请求和响应

## ASP.NET Core 中的 HttpLogging 入门

ASP.NET Core 提供了一个内置的 HttpLogging 中间件，用于记录所有传入的 Web API 请求和响应。

要启用 HTTP 日志记录，你只需要两个简单的步骤：

### 步骤 1：在 Program.cs 中添加 HttpLogging 中间件

```csharp
var builder = WebApplication.CreateBuilder(args);

// 启用HTTP日志服务
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = HttpLoggingFields.Request | HttpLoggingFields.Response;
});

var app = builder.Build();

// 将中间件添加到HTTP请求管道
app.UseHttpLogging();

app.Run();

```

### 步骤 2：在 appsettings.json 中调整日志级别

要确保 HTTP 日志记录输出出现在你的日志中，你应该调整日志记录级别。通常，将其设置为"Information"就足够了：

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Hosting.Diagnostics": "Warning",
      "Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware": "Information"
    }
  }
}
```

配置完成后，HTTP 请求和响应将自动记录到你配置的日志输出（控制台、文件等）。发出请求时，你应该会看到类似于以下示例的日志条目：

```bash
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[1]
      Request:
      Protocol: HTTP/1.1
      Method: GET
      Scheme: https
      PathBase: 
      Path: /
      Headers:
        Host: localhost:5001
        User-Agent: Mozilla/5.0...
      Response:
      StatusCode: 200
      Headers:
        Content-Type: text/plain; charset=utf-8
```

它与内置的 ASP.NET Core 日志记录提供程序一起工作 - Microsoft.Extensions.Logging 。它还可以与自定义日志记录提供程序一起使用，例如 Serilog。

Serilog 拥有庞大的接收器(sinks)生态系统，其灵活的配置使其成为日志记录的绝佳选择。接收器(sink)是你可以输出和存储日志的源；它可以是控制台、文件、数据库或监控系统。

这是配置 Serilog 以支持 HttpLogging 的方法：

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Console",
      "Serilog.Sinks.File"
    ],
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.AspNetCore.Hosting.Diagnostics": "Warning",
        "Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware": "Information"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      { "Name": "File", "Args": { "path": "service.log", "rollingInterval": "Day" } }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ],
    "Properties": {
      "Application": "ApplicationName"
    }
  }
}
```

## 配置 HttpLogging

HttpLogging 中间件可以使用 **HttpLoggingOptions** 类进行配置。

主要选项是 **HttpLoggingFields** 枚举，它控制应该记录什么内容。你可以组合多个标志以在日志中获得所需的输出。

你可以在 Microsoft 文档中查看所有可用选项。

以下是如何为 HTTP 请求配置日志记录的示例：

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = 
        HttpLoggingFields.RequestMethod |
        HttpLoggingFields.RequestPath |
        HttpLoggingFields.RequestQuery |
        HttpLoggingFields.RequestHeaders |
        HttpLoggingFields.ResponseStatusCode |
        HttpLoggingFields.Duration;
});
```

你也可以限制记录的标头：

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.RequestHeaders.Add("X-API-Version");
    options.RequestHeaders.Add("User-Agent");
    options.ResponseHeaders.Add("Content-Type");
});
```

记录请求和响应正文对于调试 API 特别有用。然而，考虑性能和安全方面至关重要：

- **性能**：记录请求和响应正文可能会减慢你的应用程序速度，因为它需要从 HTTP 正文中读取整个对象。在生产环境中请谨慎使用此选项。
- **安全与隐私**：请求体可能包含敏感用户数据（密码、信用卡信息、个人身份信息等）。请务必实施脱敏策略，或仅在安全环境（开发和测试环境）中启用请求体日志记录。

如果你的日志增长过大，你可以配置允许记录的请求/响应正文的最大大小：

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.RequestBodyLogLimit = 4096; // 单位字节
    options.ResponseBodyLogLimit = 4096;
});
```

默认值为 32 KB。

以下是如何为开发和生产环境自定义日志记录的方法：

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields =
        HttpLoggingFields.RequestMethod |
        HttpLoggingFields.RequestPath |
        HttpLoggingFields.RequestQuery |
        HttpLoggingFields.RequestHeaders |
        HttpLoggingFields.ResponseStatusCode |
        HttpLoggingFields.ResponseHeaders |
        HttpLoggingFields.Duration;

    // 指定记录这些头部信息
    options.RequestHeaders.Add("User-Agent");
    options.ResponseHeaders.Add("Content-Type");
    
    options.RequestBodyLogLimit = 2048; // 2 KB
    options.ResponseBodyLogLimit = 2048; // 2 KB

    // 安全环境下限制请求/响应体记录
    if (builder.Environment.IsDevelopment())
    {
        options.LoggingFields |= HttpLoggingFields.RequestBody | HttpLoggingFields.ResponseBody;
        options.RequestBodyLogLimit = 1024 * 32; // 32 KB
        options.ResponseBodyLogLimit = 1024 * 32; // 32 KB
    }
});
```

## 使用 IHttpLoggingInterceptor 自定义日志

你可能需要为特定端点自定义日志条目：

- 动态排除特定标头
- 不要记录特定端点的请求和响应正文
- 修改记录的数据以提高可读性或安全性

对于这些情况，你可以创建自己的 **IHttpLoggingInterceptor** 实现。这允许你在日志被写入之前拦截和自定义日志。

实现此接口使你能够：

- 动态地排除或包含特定的头信息或字段。
- 编辑请求或响应中的敏感数据。
- 向日志中添加自定义信息以提供额外的上下文。

让我们来探索一个示例：

```csharp
public class CustomLoggingInterceptor : IHttpLoggingInterceptor
{
    public ValueTask OnRequestAsync(HttpLoggingInterceptorContext context)
    {
        // 例如：删除特定的头
        context.HttpContext.Request.Headers.Remove("X-API-Key");

        // 例如：添加自定义信息到日志
        context.AddParameter("RequestId", Guid.NewGuid().ToString());

        return ValueTask.CompletedTask;
    }

    public ValueTask OnResponseAsync(HttpLoggingInterceptorContext context)
    {
        // 例如：移除敏感响应头
        logContext.HttpContext.Response.Headers.Remove("Set-Cookie");

        // 例如：添加自定义信息到日志
        context.AddParameter("new-response-field", Guid.NewGuid().ToString());

        return ValueTask.CompletedTask;
    }
}
```

在此示例中：

- **OnRequestAsync** 在请求数据被记录之前被调用，允许你自定义请求日志。
- **OnResponseAsync** 负责让你调整响应日志。

一旦你有了拦截器类，就在 Program.cs 中注册它：

```csharp
builder.Services.AddHttpLogging(loggingOptions =>
{
    // ...
});

builder.Services.AddSingleton<IHttpLoggingInterceptor, CustomLoggingInterceptor>();

var app = builder.Build();

app.UseHttpLogging();
```

## 端点特定的日志配置

通常，并非所有端点都需要相同的日志记录详细程度。ASP.NET Core 允许你按端点配置 HTTP 日志记录。

以下是一个使用 Minimal API 的示例：

```csharp
app.MapGet("/health", () => Results.Ok())
   .DisableHttpLogging(); // 不记录任何内容。

app.MapPost("/api/orders", (OrderRequest request) => 
{
    // 处理订单逻辑
    return Results.Created();
})
.WithHttpLogging(logging =>
{
    logging.LoggingFields = HttpLoggingFields.RequestBody | HttpLoggingFields.ResponseStatusCode;
});
```

## 如何在日志中屏蔽敏感数据

请记住，日志可能会捕获敏感信息，如密码、API 密钥、令牌和个人身份信息(PII)。在日志中暴露此类数据可能导致严重的安全风险和合规问题。

理解为什么需要删除敏感信息至关重要：

- **安全风险**：包含令牌、密码或 API 密钥的日志可能会使你的应用程序暴露给未授权的访问和破坏。
- **合规要求**：诸如 GDPR、HIPAA 或 PCI DSS 等法规明确要求谨慎处理和记录敏感信息。
- **信任与隐私**：维护用户隐私和信任需要主动保护个人和敏感信息。

ASP.NET Core 提供了内置功能和最佳实践，可安全地从 HTTP 日志中编辑敏感数据：

- **选择性头部日志记录**：明确指定要记录哪些头部，从而排除敏感头部。
- **使用 IHttpLoggingInterceptor**：通过自定义拦截器实现，动态删除或修改日志中的敏感数据。

以下是你如何允许仅记录特定标头的方法：

```csharp
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = 
        HttpLoggingFields.RequestMethod |
        HttpLoggingFields.RequestPath |
        HttpLoggingFields.ResponseStatusCode;

    // 只允许记录非敏感标头
    options.RequestHeaders.Add("User-Agent");
    options.ResponseHeaders.Add("Content-Type");
});
```

以下是你可以使用拦截器从日志中编辑敏感数据的方法：

```csharp
public class SensitiveDataRedactionInterceptor : IHttpLoggingInterceptor
{
    public ValueTask OnRequestAsync(HttpLoggingInterceptorContext context)
    {
        if (context.HttpContext.Request.Method == "POST")
        {
            // POST请求忽略
            context.LoggingFields = HttpLoggingFields.None;
        }

        // 如果我们不打算记录响应的任何部分，请不要扩充
        if (!context.IsAnyEnabled(HttpLoggingFields.Request))
        {
            return default;
        }

        if (context.TryDisable(HttpLoggingFields.RequestPath))
        {
            RedactPath(context);
        }

        if (context.TryDisable(HttpLoggingFields.RequestHeaders))
        {
            RedactRequestHeaders(context);
        }

        EnrichRequest(context);

        return default;
    }

    public ValueTask OnResponseAsync(HttpLoggingInterceptorContext logContext)
    {
        // 如果我们不打算记录响应的任何部分，请不要扩充
        if (!logContext.IsAnyEnabled(HttpLoggingFields.Response))
        {
            return default;
        }

        if (logContext.TryDisable(HttpLoggingFields.ResponseHeaders))
        {
            RedactResponseHeaders(logContext);
        }

        EnrichResponse(logContext);

        return default;
    }

    private static void RedactPath(HttpLoggingInterceptorContext logContext)
    {
        logContext.AddParameter(nameof(logContext.HttpContext.Request.Path), "[REDACTED]");
    }

    private static void RedactRequestHeaders(HttpLoggingInterceptorContext logContext)
    {
        foreach (var header in logContext.HttpContext.Request.Headers)
        {
            logContext.AddParameter(header.Key, "[REDACTED]");
        }
    }

    private static void EnrichRequest(HttpLoggingInterceptorContext logContext)
    {
        logContext.AddParameter("new-request-field", Guid.NewGuid().ToString());
    }

    private static void RedactResponseHeaders(HttpLoggingInterceptorContext logContext)
    {
        foreach (var header in logContext.HttpContext.Response.Headers)
        {
            logContext.AddParameter(header.Key, "[REDACTED]");
        }
    }

    private static void EnrichResponse(HttpLoggingInterceptorContext logContext)
    {
        logContext.AddParameter("new-response-field", Guid.NewGuid().ToString());
    }
}
```

让我们测试几个 HTTP 请求以及它们是如何被记录的：

```bash
GET https://localhost:5000/api/books/7dcec396-4150-4354-a51c-f2f13be3ef4c

info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[1]
      Request:
      Path: [REDACTED]
      Accept: [REDACTED]
      Host: [REDACTED]
      User-Agent: [REDACTED]
      Accept-Encoding: [REDACTED]
      Authorization: [REDACTED]
      new-request-field: 8860d646-24a3-49d3-9405-e1eccc5b7c71
      Method: GET
      QueryString:
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[2]
      Response:
      Content-Type: [REDACTED]
      new-response-field: a68d1cc0-269f-43e6-90e6-6f9a8ca178a2
      StatusCode: 200
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[4]
      ResponseBody: {"id":"7dcec396-4150-4354-a51c-f2f13be3ef4c","title":"Fish","year":2024,"authorId":"91a2d8f6-a5cf-4689-8abf-4284bc8cd5a3"}
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[8]
      Duration: 142.8386ms
```

在此示例中，整个响应正文都被记录。

而这就是 404 未找到响应可能的样子：

```bash
GET https://localhost:5000/api/books/d81b2b46-b725-4594-9465-133c443641cb

info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[1]
      Request:
      Path: [REDACTED]
      Accept: [REDACTED]
      Host: [REDACTED]
      User-Agent: [REDACTED]
      Accept-Encoding: [REDACTED]
      Authorization: [REDACTED]
      new-request-field: d856b313-668f-42a7-b8d7-3b40363375bf
      Method: GET
      QueryString:
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[2]
      Response:
      new-response-field: 3d1717f3-b23d-4689-ad09-de1b81d02cda
      StatusCode: 404
info: Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware[8]
      Duration: 134.0404ms
```

## 使用 HttpClient 发送请求时记录请求和响应

我们已经介绍了如何在 ASP.NET Core 中记录 HTTP 请求和响应。

然而，当使用 **HttpClient** 向外部服务发送请求时，理解如何记录请求和响应是至关重要的。出于可观察性和调试目的，记录与外部 API 或服务的交互是必不可少的。

你可以使用 **DelegatingHandler** 来检查来自 HttpClient 的所有 HTTP 流量。 **DelegatingHandler** 是 HttpClient 管道的一个中间件，允许你拦截和修改请求和响应。

让我们来探索这样一个处理程序的示例：

```csharp
public class HttpLoggingHandler : DelegatingHandler
{
    private readonly ILogger _logger;
    
    public HttpLoggingHandler(ILogger logger)
    {
        _logger = logger;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var traceId = request.Headers.TryGetValues("trace-id", out var values) ? values.FirstOrDefault() : null;
        traceId ??= Guid.NewGuid().ToString();

        var requestBuilder = new StringBuilder();

        var url = $"{request.RequestUri?.Host}:{request.RequestUri?.Port}{request.RequestUri?.AbsolutePath}";
        var headers = request.Headers.ExceptSensitiveHeaders().Select(x => $"[{x.Key}, {string.Join(",", x.Value)}]");

        requestBuilder.AppendLine($"[REQUEST] {traceId}");
        requestBuilder.AppendLine($"{request.Method}: {request.RequestUri?.Scheme}://{url}");
        requestBuilder.AppendLine($"Headers: {string.Join(", ", headers)}");

        if (request.Content != null)
        {
            if (request.Content.Headers.Any())
            {
                var contentHeaders = request.Content.Headers
                    .ExceptSensitiveHeaders().Select(x => $"[{x.Key}, {string.Join(",", x.Value)}]");
                
                requestBuilder.AppendLine($"Content headers: {string.Join(", ", contentHeaders)}");
            }

            if (RequestCanBeLogged(request.RequestUri?.AbsolutePath))
            {
                requestBuilder.AppendLine("Content:");
                requestBuilder.AppendLine(await request.Content.ReadAsStringAsync(cancellationToken));
            }
        }

        _logger.LogDebug("{Request}", requestBuilder.ToString());

        var stopwatch = new Stopwatch();
        stopwatch.Start();

        var response = await base.SendAsync(request, cancellationToken);
        stopwatch.Stop();

        var responseBuilder = new StringBuilder();
        responseBuilder.AppendLine($"[RESPONSE] {traceId}");
        responseBuilder.AppendLine($"{request.Method}: {request.RequestUri?.Scheme}://{url} {(int)response.StatusCode} {response.ReasonPhrase} executed in {stopwatch.Elapsed.TotalMilliseconds} ms");
        responseBuilder.AppendLine($"Headers: {string.Join(", ", response.Headers.Select(x => $"[{x.Key}, {string.Join(",", x.Value)}]"))}");

        if (response.Content.Headers.Any())
        {
            var contentHeaders = response.Content
                .Headers.Select(x => $"[{x.Key}, {string.Join(",", x.Value)}]");
            
            requestBuilder.AppendLine($"Content headers: {string.Join(", ", contentHeaders)}");
        }

        if (ResponseCanBeLogged(request.RequestUri?.AbsolutePath) && ResponseCanBeLogged(request.RequestUri?.AbsolutePath))
        {
            responseBuilder.AppendLine("Content:");
            responseBuilder.AppendLine(await response.Content.ReadAsStringAsync(cancellationToken));
        }

        _logger.LogDebug("{Response}", responseBuilder.ToString());
        _logger.LogDebug("Request completed in {ElapsedTotalMilliseconds}ms", stopwatch.Elapsed.TotalMilliseconds);
        return response;
    }
}
```

在这里，我使用以下方法来编辑数据：

1. **ExceptSensitiveHeaders** ：从日志中编辑敏感标头。
2. **RequestCanBeLogged** : 检查是否可以记录请求。
3. **ResponseCanBeLogged** : 检查响应是否可以记录。

对于 **ExceptSensitiveHeaders** ，我在 HttpRequestHeaders 和 HttpContentHeaders 上创建了一个扩展方法：

```csharp
internal static class HttpLoggingHelpers
{
    public static IEnumerable<KeyValuePair<string, IEnumerable<string>>> ExceptSensitiveHeaders(
        this HttpRequestHeaders headers)
    {
        return headers.Where(x => !x.Key.Contains("Authorization",
            StringComparison.OrdinalIgnoreCase));
    }

    public static IEnumerable<KeyValuePair<string, IEnumerable<string>>> ExceptSensitiveHeaders(
        this HttpContentHeaders headers)
    {
        return headers.Where(x => !x.Key.Contains("secret-token",
            StringComparison.OrdinalIgnoreCase));
    }
}
```

以下是检查允许 URL 的简单实现：

```csharp
private static readonly List<string> NotAllowedRequestUrls = new()
{
    "/api/users/login",
    "/api/users/refresh"
};

private static readonly List<string> NotAllowedResponseUrls = new()
{
    "/api/users/login",
    "/api/users/refresh"
};

private static bool RequestCanBeLogged(string? url)
{
    if (string.IsNullOrEmpty(url))
    {
        return false;
    }

    return !NotAllowedRequestUrls.Any(x => x.Contains(url, StringComparison.OrdinalIgnoreCase));
}

private static bool ResponseCanBeLogged(string? url)
{
    if (string.IsNullOrEmpty(url))
    {
        return false;
    }

    return !NotAllowedResponseUrls.Any(x => x.Contains(url, StringComparison.OrdinalIgnoreCase));
}
```

对于生产环境，你应该配置这些选项，但就示例而言，这已经足够了。

要使用此处理程序，你必须在配置 Program.cs 中的 HttpClient 时注册它：

```csharp
builder.Services.AddHttpClient<ITodoClient, TodoClient>(client =>
{
    client.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
})
.AddHttpMessageHandler(configure =>
{
    var logger = configure.GetRequiredService<ILoggerFactory>()
        .CreateLogger("json-placeholder-todos");
    
    return new HttpLoggingHandler(logger);
});
```

这种方法也适合 Refit 和 Polly：

```csharp
builder.Services.AddTransient<LoggingHandler>();

services
    .AddRefitClient<ITodoApi>()
    .ConfigureHttpClient((provider, c) =>
    {
        client.BaseAddress = new Uri("https://jsonplaceholder.typicode.com/");
    })
    .AddHttpMessageHandler(configure =>
    {
        var logger = configure.GetRequiredService<ILoggerFactory>()
            .CreateLogger("json-placeholder-todos");
        
        return new HttpLoggingHandler(logger);
    });
```

jsonplaceholder 是一个允许你测试 HTTP 请求的公共 API。让我们向 <https://jsonplaceholder.typicode.com/todos/50> 发送一个请求，看看日志是什么样的：

```bash
dbug: json-placeholder-todos[0]
      [REQUEST] 4d251738-5aeb-4342-824e-5d23ddf738a2
      GET: https://jsonplaceholder.typicode.com:443/todos/50
      Headers:
      
dbug: json-placeholder-todos[0]
      [RESPONSE] 4d251738-5aeb-4342-824e-5d23ddf738a2
      GET: https://jsonplaceholder.typicode.com:443/todos/50 200 OK executed in 561,7971 ms
      Headers: [Date, Tue, 10 Jun 2025 05:18:23 GMT], [Cache-Control, max-age=43200]
      Content headers: [Content-Type, application/json; charset=utf-8], [Content-Length, 121]
      Content:
      {
        "userId": 3,
        "id": 50,
        "title": "cupiditate necessitatibus ullam aut quis dolor voluptate",
        "completed": true
      }

dbug: json-placeholder-todos[0]
      Request completed in 406,5857ms
```

## 总结

日志记录对于维护健壮、安全和可维护的 ASP.NET Core 应用程序至关重要。对 HTTP 请求和响应进行适当的日志记录，使开发人员和团队能够快速诊断问题、监控应用程序性能并满足关键的合规性要求。

今天，我们探讨了 ASP.NET Core 提供的各种内置和高级技术，以实现对 HTTP 日志记录的精确控制。

**遵循这些建议，确保日志中的敏感数据安全：**

- **默认排除原则**：只记录明确安全的内容。默认情况下避免记录敏感信息。
- **环境特定日志记录**：仅在开发过程中为测试目的启用包含敏感数据的详细日志，而在生产环境中绝不应该这样做。
- **数据掩码的正则表达式**：谨慎使用正则表达式或结构化数据解析器（如 JSON 解析器）来精确掩码敏感字段。
- **测试**：通过自动化测试验证你的编辑规则，确保没有敏感数据意外泄露到日志中。

今天就到这里。期待与你再次相见。
