# [在 ASP.NET Core 中通过用户上下文增强请求跟踪](https://www.milanjovanovic.tech/blog/better-request-tracing-with-user-context-in-asp-net-core)

在构建 Web 应用程序时，了解幕后发生的情况至关重要。在 ASP.NET Core 中，我们可以通过将用户上下文添加到请求跟踪中，使我们的生活变得更轻松。这有助于我们跟踪问题、理解用户行为并改进我们的应用程序。

让我向你展示如何在 ASP.NET Core 应用程序中通过添加用户上下文来增强请求跟踪。

## 为什么要在请求跟踪中添加用户上下文？

当你的应用程序出现问题时，在日志中拥有用户的 ID 可以大大简化问题排查过程。你无需在成千上万的日志条目中搜索，只需按用户 ID 过滤，即可查看相关的日志。

添加用户上下文还有助于：

- 通过应用程序跟踪用户旅程
- 识别用户行为模式
- 特定用户的故障排除
- 分段性能监控不同用户的

## 实现用户上下文增强

我们解决方案的核心是一个中间件组件，它从当前用户的声明中提取用户 ID，并将其添加到当前活动和日志记录作用域中。

ASP.NET Core 中的**日志作用域**允许你将附加数据附加到在该作用域内创建的所有日志消息中。像 Serilog 和内置日志器这样的结构化日志框架支持此功能。例如，如果你将用户 ID 添加到日志作用域中，该作用域内的每条日志消息都将包含该用户 ID，即使日志消息本身没有提及用户。这使得跨应用程序的不同部分轻松关联特定用户的日志变得容易。

**Activity** 类是 .NET 诊断基础设施的一部分。它表示一个工作单元或操作，旨在跨服务边界进行分布式跟踪。当你向 **Activity.Current** 添加一个标签时，该信息将成为跟踪的一部分，并可用于在你的监控系统中过滤和分析请求。

以下是代码：

```csharp
using System.Diagnostics;
using System.Security.Claims;

namespace MyApp.Middleware;

public sealed class UserContextEnrichmentMiddleware(
    RequestDelegate next,
    ILogger<UserContextEnrichmentMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        string? userId = context.User?.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is not null)
        {
            Activity.Current?.SetTag("user.id", userId);

            var data = new Dictionary<string, object>
            {
                ["UserId"] = userId
            };

            using (logger.BeginScope(data))
            {
                await next(context);
            }
        }
        else
        {
            await next(context);
        }
    }
}
```

让我们分解一下这个中间件的作用：

1. 它从已认证用户的声明中提取用户 ID，使用 context.User?.FindFirstValue(ClaimTypes.NameIdentifier)
2. 如果找到用户 ID，它会将该 ID 作为标签添加到当前活动中，使用 Activity.Current?.SetTag("user.id", userId)
3. 它使用 ILogger.BeginScope 和用户 ID 创建一个日志作用域，这意味着该作用域内的所有日志条目都将包含用户 ID
4. 它调用管道中的下一个中间件
5. 如果未找到用户 ID（对于匿名请求），它将直接调用下一个中间件

## 日志作用域和 OpenTelemetry

如果你正在使用 OpenTelemetry 来导出日志，你必须配置提供程序以在生成的日志记录中包含日志范围。

以下是方法：

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeScopes = true;
    options.IncludeFormattedMessage = true;
});
```

## 将中间件添加到你的应用程序

在你的 ASP.NET Core 应用程序中使用此中间件，请在 Program.cs 文件中将其添加到你的管道中：

```csharp
app.UseAuthentication();
app.UseAuthorization();

// 确保将其放置在身份验证和授权中间件之后，以便可以使用用户身份。
app.UseMiddleware<UserContextEnrichmentMiddleware>();

app.MapControllers();

app.Run();
```

确保将其放置在身份验证和授权中间件之后，以便可以使用用户身份。

## 处理个人身份信息（PII）问题

在将用户 ID 添加到日志和跟踪时，你需要小心处理个人可识别信息（PII）。以下是一些需要考虑的重要事项：

- 用户 ID 应该是不可透明的标识符（如 GUID），不应透露个人信息
- 避免记录电子邮件地址、姓名或其他个人数据
- 确保你的日志配置不会将个人识别信息（PII）发送到不应发送的系统

如果你需要遵守数据保护法规，请考虑实施日志保留策略，并在需要时从日志中清除用户数据的能力。

## 扩展上下文增强

用户 ID 只是开始。我们可以添加更多上下文，使我们的日志和跟踪更加有用：

### 特性标志

**特性标志**帮助我们逐步推出新功能或为特定用户启用它们。将特性标志信息添加到我们的上下文中，为我们提供了宝贵的洞察：

```csharp
// 置于中间件内部
if (featureFlagService.IsEnabled("NewFeature", userId))
{
    Activity.Current?.SetTag("features.newfeature", "enabled");

    // TODO: 添加到日志作用域中
}
```

### 多租户应用的租户信息

如果你的应用程序服务于多个租户，添加租户 ID 会非常有帮助：

```csharp
string? tenantId = context.User?.FindFirstValue("TenantId");
if (tenantId is not null)
{
    Activity.Current?.SetTag("tenant.id", tenantId);
    
    // TODO: 添加到日志作用域中
}
```

## 总结

在 ASP.NET Core 中为请求跟踪添加用户上下文是一种简单而强大的技术。通过实现我们探讨过的中间件，你将看到几个重要的好处：

1. 更快的故障排除 - 当用户报告问题时，你可以快速找到相关日志
2. 更好地理解使用模式 - 查看哪些功能正在被使用以及使用者的身份
3. 改进性能监控 - 识别特定用户段的慢请求
4. 更有效的 A/B 测试 - 跟踪具有不同功能标志的用户指标

了解日志作用域和 Activity 类有助于你充分利用这一技术。日志作用域确保你的日志条目包含一致的上下文信息，而活动则支持跨服务边界的分布式追踪。请务必小心处理个人识别信息（PII），并确保你的日志实践符合相关法规。

今天就到这里。期待与你再次相见。
