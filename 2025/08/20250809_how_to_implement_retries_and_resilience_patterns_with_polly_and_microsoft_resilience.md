# [如何使用 Polly 和 Microsoft Resilience 实现重试和弹性模式](https://antondevtips.com/blog/how-to-implement-retries-and-resilience-patterns-with-polly-and-microsoft-resilience)

现代 .NET 应用程序通常依赖外部服务来获取数据、消息传递等。如果处理不当，单个网络错误可能导致级联故障。

现代 在调用外部 API 时，您可能会遇到以下几个问题：.NET 应用程序通常依赖外部服务来获取数据、消息传递等。如果处理不当，单个网络错误可能导致级联故障。

- 瞬时故障：外部服务可能会在短时间内变慢或无法访问。
- 部分故障：另一个系统可能处于维护模式，限制了您的访问。
- 网络不稳定：网络连接较慢的用户可能会遇到短时间的中断或超时。
- 过载服务：另一个服务可能因请求过多而过载，导致响应缓慢。

网络是不稳定的，你迟早会遇到这些问题。

这句话改变了我对构建弹性系统的思考方式："系统的弹性不在于其缺乏错误，而在于其能够承受许多错误的能力。"

这就是**弹性模式**的用武之地。与其立即失败，不如给这些服务第二次机会来重试或将调用路由到备用服务，从而提高应用程序的健壮性和稳定性。

今天我想向大家展示如何使用 **Polly** 和 **Microsoft.Extensions.Resilience** 包来构建弹性应用程序。

让我们开始吧。

## Polly 和 Microsoft.Resilience 入门指南

让我们来探索两个服务：订单和支付。当用户订单创建后，下一步就是执行支付。

```csharp
builder.Services.AddHttpClient("PaymentsClient", client =>
{
    client.BaseAddress = new Uri("http://localhost:5221");
    client.Timeout = TimeSpan.FromSeconds(60);
});

var client = clientFactory.CreateClient("PaymentsClient");
var response = await client.PostAsync("/api/payments/create", null);
response.EnsureSuccessStatusCode();
```

这对于确保当用户付款时，支付请求能够正确处理且不出现任何问题至关重要。

有 5 种主要的弹性模式：

- 重试
- 断路器
- 超时
- 降级
- 对冲

在.NET 应用程序中，这些弹性模式是借助两个库来实现的：

- Polly
- Microsoft.Extensions.Resilience

**Polly** 是一个经典的.NET 库，多年来广为人知，它能帮助您以最小的努力实现弹性模式。它允许围绕任何代码创建弹性管道：网络调用、数据库、缓存、发送电子邮件等。

**Microsoft.Extensions.Resilience** 构建于 Polly 之上。它提供了与 .NET HttpClientFactory 的无缝集成。

这使您能够将弹性管道应用于外部 HTTP 调用。

将以下 Nuget 包添加到订单服务：

```bash
dotnet add package Polly
dotnet add package Microsoft.Extensions.Resilience
```

现在让我们深入了解每个弹性模式。

## 实现重试模式

当服务调用因网络错误而失败时，开发人员通常只在屏幕上显示一条通用错误消息："请稍后再试。"与快速失败不同，通过实施重试可以在抛出错误之前给服务另一个恢复的机会，从而提高系统的弹性。

以下是主要的重试类型：

- **线性重试**：在每次尝试之间等待固定的延迟时间后进行重试。
- **指数重试**：在每次尝试之间以指数级增长的延迟进行重试。
- **随机重试**：在重试之间添加随机延迟，以分散负载并避免重试聚集。
- **带抖动的指数退避**：将延迟的指数增长与随机抖动相结合，以避免重试风暴。

**带抖动的指数退避**被认为是最可靠的重试策略。幸运的是，你不需要自己实现它，Polly 已经为此提供了内置的管道。

这是您可以定义重试管道的方法：

```csharp
using Polly;
using Polly.Retry;

var retryOptions = new RetryStrategyOptions
{
    ShouldHandle = new PredicateBuilder()
        .Handle<HttpRequestException>(),
        
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,
    MaxRetryAttempts = 5,
    Delay = TimeSpan.FromSeconds(3),
};

var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(retryOptions)
    .Build();
```

现在你可以包装任何将自动重试的代码。以下是如何在 HttpClient 调用中使用它：

```csharp
var result = await pipeline.ExecuteAsync(async token =>
{
    var response = await httpClient.PostAsync("/api/payments/create");
    response.EnsureSuccessStatusCode(); 
    return await response.Content.ReadAsStringAsync();
}, cancellationToken);
```

**Microsoft.Extensions.Resilience** 提供了一种更便捷的方式来为所有 HttpClient 调用应用重试：

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;
using Polly.Retry;

builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
})
.AddResilienceHandler("RetryStrategy", resilienceBuilder =>
{
    resilienceBuilder.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 4,
        Delay = TimeSpan.FromSeconds(3),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(response => !response.IsSuccessStatusCode)
    });
});
```

当通过给定的 HttpClient 发送任何请求时，重试管道将由在后台创建的 **DelegateHandler** 自动调用。

拥有重试机制总是有益的，因为它可以避免由瞬时网络问题引起的不必要失败。

## 实现断路器模式

想象一下，一个服务不仅仅是短暂不可用，而是一直在失败。无休止的重试可能会使情况变得更糟。断路器在检测到大量失败时会切断请求，立即返回错误，直到冷却期结束。

断路器的工作原理类似于电路。

**它具有以下状态**：

- **关闭状态**：一切正常；网络调用正常通过。
- **打开状态**：在出现过多失败后，断路器"打开"并在设定的时间内阻止调用。它会立即返回错误。
- **半开状态**：当断路时间结束时，它会测试几次调用以查看服务是否已恢复正常。如果成功，它将转换回关闭状态；如果不成功，则保持开启状态。

以下是定义断路器管道的方法：

```csharp
using Polly;
using Polly.CircuitBreaker;

var options = new CircuitBreakerStrategyOptions
{
    FailureRatio = 0.5,
    SamplingDuration = TimeSpan.FromSeconds(10),
    MinimumThroughput = 8,
    BreakDuration = TimeSpan.FromSeconds(30),
    ShouldHandle = new PredicateBuilder()
        .Handle<HttpRequestException>()
};

var pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(options)
    .Build();
```

它具有以下选项：

- **FailureRatio**：触发断路器的失败调用百分比。
- **SamplingDuration**：用于监控调用失败的时间窗口。
- **MinimumThroughput**：断路器采取行动所需的最小调用次数。
- **BreakDuration**：断路器在尝试再次关闭之前保持打开状态的持续时间。

让我们在 HttpClient 上测试我们的断路器：

```csharp
for (int i = 0; i < 10; i++)
{
    var result = await pipeline.ExecuteAsync(async token =>
    {
        var response = await httpClient.PostAsync("/api/payments/create");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }, cancellationToken);
}
```

**Microsoft.Extensions.Resilience** 提供了一种更便捷的方式为所有 HttpClient 调用添加断路器：

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;
using Polly.CircuitBreaker;

builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
})
.AddResilienceHandler("CurcuitBreakerStrategy", resilienceBuilder =>
{
    resilienceBuilder.AddCircuitBreaker(
        new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            MinimumThroughput = 8,
            BreakDuration = TimeSpan.FromSeconds(30),
            ShouldHandle = static args => ValueTask.FromResult(args is
            {
                Outcome.Result.StatusCode:
                HttpStatusCode.RequestTimeout or
                HttpStatusCode.TooManyRequests
            })
        });
});
```

断路器允许在外部服务不可用时快速失败，并在服务恢复在线时迅速恢复。

## 实现超时模式

有些服务响应速度可能非常快，而其他服务则可能非常缓慢，甚至无限期挂起。

这就是为什么你需要为你的网络请求实现超时机制。

超时模式确保您的请求以受控方式失败。

**最佳实践**：

- 为你的 HttpClient 设置全局超时（例如，30-60 秒）。
- 在每次重试中，设置一个较小的"每次尝试"超时时间，这样就不会每次等待太久。

超时模式对于 HttpClient 调用尤其重要。以下是设置方法：

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;

builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
    client.Timeout = TimeSpan.FromSeconds(60); // 全局超时设定
})
.AddResilienceHandler("TimeoutStrategy", resilienceBuilder =>
{
    resilienceBuilder
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 4,
            Delay = TimeSpan.FromSeconds(3),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .HandleResult(response => !response.IsSuccessStatusCode)
        })
        .AddTimeout(TimeSpan.FromSeconds(10)); // 每次尝试的超时设定
});
```

## 实现降级模式

如果所有重试都失败了怎么办？降级模式提供了一种优雅的方式来处理最坏情况。

它允许在所有重试失败时执行某个操作。这对于紧急例程非常有用：记录错误、发送警报或返回缓存数据。

通常，回退模式与重试模式一起使用。以下是创建方法：

```csharp
using Polly;
using Polly.Fallback;
using Polly.Retry;

var retryOptions = new RetryStrategyOptions<HttpResponseMessage>
{
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        .Handle<HttpRequestException>(),
    
    BackoffType = DelayBackoffType.Exponential,
    UseJitter = true,
    MaxRetryAttempts = 5,
    Delay = TimeSpan.FromSeconds(3),
};

var fallbackOptions = new FallbackStrategyOptions<HttpResponseMessage>
{
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>().Handle<ApplicationException>(),
    FallbackAction = args =>
    {
        Console.WriteLine("All retries failed. Sending alert email...");
        return Outcome.FromResultAsValueTask(
            new HttpResponseMessage(HttpStatusCode.InternalServerError));
    }
};
```

以下是如何结合重试和回退管道，并用组合管道包装 HttpClient 调用的方法：

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddFallback(fallbackOptions)
    .AddRetry(retryOptions)
    .Build();

var result = await pipeline.ExecuteAsync(async token =>
{
    var response = await httpClient.PostAsync("/api/payments/create", token);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
}, CancellationToken.None);
```

以下是如何为 HttpClient 配置降级策略：

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;
using Polly.Fallback;
using Polly.Retry;

builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
})
.AddResilienceHandler("RetryFallbackStrategy", resilienceBuilder =>
{
    resilienceBuilder
        .AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(2),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .HandleResult(response => !response.IsSuccessStatusCode)
        })
        .AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
        {
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .Handle<HttpRequestException>()
                .HandleResult(response => !response.IsSuccessStatusCode),
            
            FallbackAction = args =>
            {
                Console.WriteLine("All retries failed. Sending alert email...");
                return Outcome.FromResultAsValueTask(
                    new HttpResponseMessage(HttpStatusCode.InternalServerError));
            }
        });
});
```

## 实现对冲模式

对冲是一种不太为人所知的模式，当原始调用耗时过长时，它会发送并行请求。第一个成功的响应获胜，其余调用则被取消。

使用对冲策略而不是等待，你的系统会向同一服务、另一个服务或副本发送备份请求。这确保了更快的响应和更高的可靠性，代价是额外的资源使用。

当您绝对需要更低的延迟且能够承受额外开销时，请使用对冲策略。

以下是实现对冲管道的方法：

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;
using Polly.Hedging;

builder.Services.AddHttpClient("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
})
.AddResilienceHandler("HedgingStrategy", resilienceBuilder =>
{
    resilienceBuilder.AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
    {
        MaxHedgedAttempts = 3, // 首个请求 + 2 个额外请求
        Delay = TimeSpan.FromSeconds(1), // 等待后再发送下一个尝试
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(response => !response.IsSuccessStatusCode)
            .Handle<HttpRequestException>(),
        
        // 订阅对冲事件
        OnHedging = args =>
        {
            Console.WriteLine("Hedging request sent due to slow response.");
            return ValueTask.CompletedTask;
        },
        ActionGenerator = static args =>
        {
            Console.WriteLine("Preparing to execute hedged action.");
    
            // 返回一个委托函数来调用原始操作，并传入操作上下文。
            // 你也可以选择创建一个全新的操作来执行。
            return () => args.Callback(args.ActionContext);
        }
    });
});
```

## 标准弹性管道

对于许多应用程序来说，**Microsoft.Extensions.Resilience** 提供了一个内置的"标准"管道，它将少数策略组合成单一链。

此管道包含以下弹性策略：

- **速率限制器**：限制可以发送到服务的并发请求数量。
- **全局请求超时**：为操作引入总体时间限制，包括所有重试。
- **重试**：自动重试因网络错误而失败的调用。
- **断路器**：一旦故障率过高，会暂时阻止请求。
- **尝试超时**：为每个单独的请求尝试设置超时。

```csharp
services.AddHttpClient<GitHubService>("PaymentService", client =>
{
    client.BaseAddress = new Uri("https://azure.paymentservice");
})
.AddStandardResilienceHandler();
```

所有这些策略都可以通过 **HttpStandardResilienceOptions** 进行配置。

## 总结

构建有弹性的 .NET 应用程序意味着优雅地处理外部故障。Polly 和 Microsoft.Extensions.Resilience 帮助您实现：

- **重试**：给服务第二次响应的机会。
- **断路器**：停止调用无响应的服务，以防止级联故障。
- **超时**：在挂起的请求消耗过多资源之前将其切断。
- **降级**：当其他所有方法都失败时，提供一个备用方案或发出警报。
- **对冲**：在延迟敏感场景中，通过发起多个请求竞赛来提高速度。

通过将这些模式集成到您的 HttpClient 或自定义管道中，您可以构建强大且可靠的.NET 应用程序，使其在出现错误时保持稳定。

今天就到这里。期待与你再次相见。
