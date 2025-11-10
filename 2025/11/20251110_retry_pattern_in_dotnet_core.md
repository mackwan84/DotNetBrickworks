# [.NET Core 中的重试模式](https://blog.devgenius.io/retry-pattern-in-net-core-adc2b84ec5aa)

在基于 Web 的应用程序中，我们连接到 Azure 数据库、Azure 保管库、Azure Cosmos DB、Azure 函数等，可能会遇到"服务器无响应"之类的临时错误。

## 我们可以将这些临时问题称为瞬时故障

在云计算领域，我们可能因各种原因导致连接失败，而这些问题通常会在几秒钟内自动恢复。因此，与其向客户端返回连接错误，我们应该尝试使用定义的策略重新连接 Azure 资源。

## 瞬时故障的可能原因

- 资源过载：例如，在高流量场景中，有限的系统资源（如内存或 CPU）可能导致尝试访问这些资源时出现故障。
- 网络问题：临时性问题如 DNS 解析错误、连接中断或网络拥塞可能导致操作失败。
- 服务不可用：外部服务或数据库可能暂时宕机或不可用。例如，数据库连接可能因为服务器暂时过载而失败。
- 超时：由于服务器过载或高网络延迟，请求可能需要比预期更长的时间来响应。
- 速率限制：一些外部 API 或服务可能会施加速率限制，如果在短时间内发出过多请求，会导致请求被暂时拒绝。

## 解决方案

今天，我们将学习重试模式，因此当我们遇到连接错误时，我们将尝试重新连接，并查看应用程序是否开始响应。

### 重试模式

重试模式会自动重试失败的操作，因为如果再次尝试，可能会成功。

![我们将在尝试 3 次后响应客户端](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*JlkxVpuCBZcFgaqv.png)

### 如何实现

我们可以创建策略并配置想要重试的次数以及间隔的秒数

```csharp
private static readonly AsyncRetryPolicy _retryPolicy = Policy
        .Handle<HttpRequestException>()
        .Or<TaskCanceledException>()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
```

代码示例

```csharp
using Polly;
using Polly.Retry;
using System;
using System.Net.Http;
using System.Threading.Tasks;

public class RetryExample
{
    private static readonly HttpClient _httpClient = new HttpClient();

    private static readonly AsyncRetryPolicy _retryPolicy = Policy
        .Handle<HttpRequestException>()
        .Or<TaskCanceledException>()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

    public static async Task Main(string[] args)
    {
        try
        {
            var response = await _retryPolicy.ExecuteAsync(() => MakeRequest());
            Console.WriteLine($"Response: {response}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Request failed: {ex.Message}");
        }
    }

    private static async Task<string> MakeRequest()
    {
        var response = await _httpClient.GetAsync("https://example.com");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```

解释说明

- 重试策略配置为处理 HttpRequestException 和 TaskCanceledException 。
- 该操作最多重试 3 次，采用指数退避策略（2、4 和 8 秒）。
- 如果在重试次数内操作成功，则处理响应；否则，记录异常。

## 总结

重试模式对于处理瞬时故障非常有用，尤其是在与外部服务交互时。通过实现重试逻辑，我们可以提高应用程序的可靠性和用户体验。

今天就到这里。希望这些内容对你有帮助。
