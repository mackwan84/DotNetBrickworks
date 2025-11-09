# [C# 中的异步编程是什么？](https://dometrain.com/blog/what-is-asynchronous-programming-in-csharp/)

异步代码让你的应用程序能够处理更多工作而不会阻塞线程。在 C# 中，async/await 是编写非阻塞 I/O 的方式，同时代码仍然具有良好的可读性。本文涵盖了基础知识、你每天都会使用的核心 API、需要避免的错误，以及一些与 ASP.NET Core 配合良好的模式。

## 思维模型

- 任务不是线程。 Task 是对结果的承诺。对于 I/O 操作，当数据到达时操作系统会唤醒你的任务，而不会有线程被阻塞等待。
- async / await 是组合，而不是魔法。 await 注册一个延续并将控制权返回给调用者，直到被等待的工作完成。在幕后，会生成一个状态机来为你处理所有这些事情。
- I/O 密集型与 CPU 密集型。异步适用于 I/O（HTTP、数据库、文件、套接字）。仅将 Task.Run 用于短暂的 CPU 工作；你不想阻塞请求线程。

## 最小 API：将 CancellationToken 一直传递下去

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/weather/{city}", async Task<IResult> (string city, CancellationToken ct) =>
{
using var http = new HttpClient();
http.Timeout = TimeSpan.FromSeconds(5);

    var json = await http.GetStringAsync($"https://example.com/weather/{city}", ct);
    return Results.Text(json, "application/json");
});

app.Run();
```

- ASP.NET Core 提供了一个 CancellationToken ，当客户端断开连接或发生超时时会触发。
- 将其作为参数并转发到每个异步调用中。当不再需要请求时释放容量。

## 永远不要在异步代码上阻塞

这些模式在服务器代码中会带来麻烦：

```csharp
// 错误的：阻塞异步代码
var client = new HttpClient();
var response = client.GetStringAsync("https://example.com").Result;
// 或者
var response = client.GetStringAsync("https://example.com").GetAwaiter().GetResult();
```

应该这样做：

```csharp
// 正确的：异步等待
var client = new HttpClient();
var response = await client.GetStringAsync("https://example.com");
```

尽管 ASP.NET Core 没有传统的 UI 同步上下文，但阻塞会占用运行时可用于处理其他请求的线程。

## 使用 Task.WhenAll 组合多个调用

```csharp
using var http = new HttpClient();

Task<string> u1 = http.GetStringAsync("https://api.test/users/1");
Task<string> u2 = http.GetStringAsync("https://api.test/users/2");
Task<string> u3 = http.GetStringAsync("https://api.test/users/3");

var results = await Task.WhenAll(u1, u2, u3);
```

- 先发起请求，然后一起等待它们。
- 这比依次等待每个请求要快得多。
- 错误： WhenAll 抛出第一个异常；如果需要，你可以检查 Task.Exception 来查看聚合异常。

## 限制工作： SemaphoreSlim 或 Parallel.ForEachAsync

一次性调用 API 1,000 次是不礼貌的，而且通常没有意义。请限制并发数量。

```csharp
var urls = Enumerable.Range(1, 1000).Select(i => $"https://api.test/data/{i}");
using var http = new HttpClient();
using var gate = new SemaphoreSlim(10);

var tasks = urls.Select(async url =>
{
    await gate.WaitAsync();
    try
    {
        return await http.GetStringAsync(url);
    }
    finally
    {
        gate.Release();
    }
});

var data = await Task.WhenAll(tasks);
```

或者使用内置的辅助方法：

```csharp
await Parallel.ForEachAsync(urls, new ParallelOptions { MaxDegreeOfParallelism = 10 }, async (url, ct) =>
{
    using var http = new HttpClient();
    await http.GetStringAsync(url, ct);
});
```

## 不使用 Thread.Sleep 的超时： WaitAsync

为任何任务添加超时：

```csharp
var call = http.GetStringAsync("https://api.test/slow", ct);
var withTimeout = call.WaitAsync(TimeSpan.FromSeconds(2), ct);
var payload = await withTimeout; // 2秒后抛出异常
```

Task.WaitAsync （适用于.NET 6 及以上版本）可以清晰地组合超时操作，无需管理计时器。

## 你会重用的取消模式

用于单次调用超时的链接令牌：

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
cts.CancelAfter(TimeSpan.FromSeconds(3));
var json = await http.GetStringAsync(url, cts.Token);
```

优雅的后台工作循环：

```csharp
public sealed class CleanupWorker(IServiceScopeFactory scopes, ILogger<CleanupWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
        while (await timer.WaitForNextTickAsync(ct))
        {
            using var scope = scopes.CreateScope();
            var svc = scope.ServiceProvider.GetRequiredService<ICleanupService>();
            await svc.RunAsync(ct);
        }
    }
}
```

## 异步流： IAsyncEnumerable<T>

在结果到达时进行流式处理，以保持低内存占用并更早开始发送数据。

```csharp
app.MapGet("/ticks", (CancellationToken ct) => GetTicks(ct));

static async IAsyncEnumerable<string> GetTicks([EnumeratorCancellation] CancellationToken ct)
{
    for (var i = 0; i < 5; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(500, ct);
        yield return DateTime.UtcNow.ToString("O");
    }
}
```

使用 await foreach 消费：

```csharp
await foreach (var tick in GetTicks(ct))
{
    Console.WriteLine(tick);
}
```

## ValueTask 正确的方式

ValueTask 当结果通常已经可用时，可以避免分配。不要默认返回它；只在已测量到性能提升的热路径中使用它。

```csharp
public interface ICache
{
    ValueTask<string?> TryGetAsync(string key, CancellationToken ct);
}

public sealed class MemoryFirstCache(IMemoryCache cache, HttpClient http) : ICache
{
    public ValueTask<string?> TryGetAsync(string key, CancellationToken ct)
    {
        if (cache.TryGetValue(key, out string? hit))
            return ValueTask.FromResult(hit); // 避免分配

        return new ValueTask<string?>(FetchAndStoreAsync(key, ct));
    }

    private async Task<string?> FetchAndStoreAsync(string key, CancellationToken ct)
    {
        var value = await http.GetStringAsync($"https://api.test/data/{key}", ct);
        cache.Set(key, value, TimeSpan.FromMinutes(1));
        return value;
    }
}
```

经验法则：

- 不要存储 ValueTask 以备后用。
- 不要多次等待它。
- 如果你不需要它的好处，坚持使用 Task 。

## 库代码和 ConfigureAwait(false)

在应用程序代码（ASP.NET Core）中，你通常不需要 ConfigureAwait(false) 。在可能被任何地方使用的库中（包括 UI 应用程序），添加它可以避免捕获调用者可能拥有的同步上下文：

```csharp
public async Task<string> LoadAsync(string path, CancellationToken ct)
{
    using var fs = File.OpenRead(path);
    using var sr = new StreamReader(fs);
    return await sr.ReadToEndAsync(ct).ConfigureAwait(false);
}
```

自从 .NET Core 发布以来，我们不再有同步上下文，因此 ConfigureAwait(false) 变得不再那么重要。

## 常见错误

- 忘记传递 CancellationToken 。在端点中接受它并向下传递。
- 使用 .Result / .Wait() 阻塞异步操作。保持整个调用链的异步性。
- 过度使用 Task.Run 。它不会使 I/O 操作更快；只是浪费一个线程。
- 忽略背压。使用 SemaphoreSlim 或 Parallel.ForEachAsync 进行限流。
- 即发即弃且不加保护。如果必须这样做，请捕获异常并记录它们，或将工作排队到后台服务。

## 实用代码片段

超时助手：

```csharp
public static class TaskExt
{
    public static Task<T> WithTimeout<T>(this Task<T> task, TimeSpan timeout, CancellationToken ct = default) =>
        task.WaitAsync(timeout, ct);
}
```

重试并取消（不使用 Polly）：

```csharp
public static async Task<T> RetryAsync<T>(Func<CancellationToken, Task<T>> action, int maxAttempts, TimeSpan delay, CancellationToken ct)
{
    for (var attempt = 1; ; attempt++)
    {
        try 
        {
            return await action(ct);
        }
        catch when (attempt < maxAttempts)
        {
            await Task.Delay(delay, ct);
        }
    }
}
```

## 总结

异步并不是让所有内容并行运行；而是在操作系统执行 I/O 操作时不浪费线程。保持异步链，传递取消令牌，使用 WhenAll 进行组合，并在需要时进行限流。

今天就到这里。希望这些内容对你有帮助。
