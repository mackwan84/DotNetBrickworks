# [如何正确地在 ASP.NET Core 中构建异步 API](https://www.milanjovanovic.tech/blog/building-async-apis-in-aspnetcore-the-right-way)

大多数 API 都遵循一个简单的模式。客户端发送请求，服务器执行一些工作，然后服务器发回响应。

这对于快速操作（如获取数据或简单更新）效果很好。但对于耗时较长的操作呢？

想想处理大文件、生成报告或转换视频。这些操作可能需要几分钟甚至几小时。

让客户端等待这些操作会导致问题。

## 理解异步 API

处理长时间运行操作的关键在于改变我们对 API 响应的思考方式。异步 API 将工作分为两部分：

- 接受请求
- 稍后处理

首先，我们接受请求并立即返回一个跟踪 ID。这给用户一个快速的响应。然后，我们在后台处理实际工作，这不会阻塞其他请求。用户可以随时使用跟踪 ID 检查他们请求的状态。

这与 C# 中的 async / await 不同。那是关于同时处理多个请求（并发）。这是关于更好地处理长时间运行的任务。我们不仅仅是让代码异步化 - 我们是从用户的角度让整个操作异步化。

## 同步 API 的问题

让我们通过图像处理来看看实际应用。一个典型的图像上传 API 可能如下所示：

```csharp
[HttpPost]
public async Task<IActionResult> UploadImage(IFormFile file)
{
    if (file is null)
    {
        return BadRequest();
    }

    // 保存原始图像
    var originalPath = await SaveOriginalAsync(file);

    // 生成缩略图
    var thumbnails = await GenerateThumbnailsAsync(originalPath);

    // 优化图像
    await OptimizeImagesAsync(originalPath, thumbnails);

    return Ok(new { originalPath, thumbnails });
}
```

客户端必须等待我们保存文件、生成缩略图和优化图像。在连接缓慢或文件较大的情况下，此请求可能会超时。服务器也被困在一次只能处理一张图像的状态中。

![API 请求](https://www.milanjovanovic.tech/blogs/mnw_117/sync_api_request.png?imwidth=3840)

## 更好的方法：异步处理

让我们解决这些问题。我们将工作分为两部分：

- 接受上传并快速返回
- 在后台执行繁重工作

![API 请求](https://www.milanjovanovic.tech/blogs/mnw_117/async_api_request.png?imwidth=3840)

### 上传图片

这是新的上传端点：

```csharp
[HttpPost]
public async Task<IActionResult> UploadImage(IFormFile? file)
{
    if (file is null)
    {
        return BadRequest("No file uploaded.");
    }

    if (!imageService.IsValidImage(file))
    {
        return BadRequest("Invalid image file.");
    }

    // 第一阶段：保存原始图像
    var id = Guid.NewGuid().ToString();
    var folderPath = Path.Combine(_uploadDirectory, "images", id);
    var fileName = $"{id}{Path.GetExtension(file.FileName)}";
    var originalPath = await imageService.SaveOriginalImageAsync(
        file,
        folderPath,
        fileName
    );

    // 第二阶段：排队后台处理
    var job = new ImageProcessingJob(id, originalPath, folderPath);
    await jobQueue.EnqueueAsync(job);

    // 立即返回状态 URL
    var statusUrl = GetStatusUrl(id);
    return Accepted(statusUrl, new { id, status = "queued" });
}
```

这个新版本在 HTTP 请求期间只保存原始文件。繁重的工作转移到了后台进程。客户端立即在 Location 标头中获取状态 URL，而无需等待。

### 检查进度

客户端可以使用单独的端点检查其图像状态：

```csharp
[HttpGet("{id}/status")]
public IActionResult GetStatus(string id)
{
    if (!statusTracker.TryGetStatus(id, out var status))
    {
        return NotFound();
    }

    var response = new
    {
        id,
        status,
        links = new Dictionary<string, string>()
    };

    if (status == "completed")
    {
        response.links = new Dictionary<string, string>
        {
            ["original"] = GetImageUrl(id),
            ["thumbnail"] = GetThumbnailUrl(id, width: 200),
            ["preview"] = GetThumbnailUrl(id, width: 800)
        };
    }

    return Ok(response);
}
```

### 在后台处理图像

真正的工作发生在后台处理器中。当 API 处理新请求时，一个单独的进程会处理队列中的作业。这种分离使我们在处理方式上具有灵活性。

对于单服务器部署，我们可以使用.NET 的 Channel 类型在内存中排队作业：

```csharp
public class JobQueue
{
    private readonly Channel<ImageProcessingJob> _channel;

    public JobQueue()
    {
        var options = new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait
        };
        _channel = Channel.CreateBounded<ImageProcessingJob>(options);
    }

    public async ValueTask EnqueueAsync(ImageProcessingJob job,
        CancellationToken ct = default)
    {
        await _channel.Writer.WriteAsync(job, ct);
    }

    public IAsyncEnumerable<ImageProcessingJob> DequeueAsync(
        CancellationToken ct = default)
    {
        return _channel.Reader.ReadAllAsync(ct);
    }
}
```

对于多服务器设置，我们需要一个分布式队列，比如 RabbitMQ 或者是 Redis。

后台处理器处理耗时的工作：

```csharp
public class ImageProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in jobQueue.DequeueAsync(ct))
        {
            try
            {
                await statusTracker.SetStatusAsync(
                    job.Id,
                    "processing"
                );

                // 生成缩略图
                await GenerateThumbnailsAsync(
                    job.OriginalPath,
                    job.OutputPath
                );

                // 优化图像
                await OptimizeImagesAsync(
                    job.OriginalPath,
                    job.OutputPath
                );

                await statusTracker.SetStatusAsync(
                    job.Id,
                    "completed"
                );
            }
            catch (Exception ex)
            {
                await statusTracker.SetStatusAsync(
                    job.Id,
                    "failed"
                );

                logger.LogError(ex, "Failed to process image {Id}", job.Id);
            }
        }
    }
}
```

后台处理器需要优雅地处理故障。我们可以通过使用 Polly 添加重试策略来提高弹性。状态更新在整个过程中让用户保持知情。我们不只是告诉他们"正在处理"，而是准确地告知他们正在发生什么。这改善了用户体验并有助于调试。

## 超越轮询：实时更新

我们的状态端点虽然可用，但它将负担转移给了客户端。客户端必须反复检查更新，导致不必要的服务器负载。每秒轮询一次的客户端每分钟会产生 60 个请求，而其中大多数请求返回的是相同的状态。

我们可以改变这种模式。不是客户端请求更新，而是服务器在更新发生时主动推送。这创建了一个更高效、响应更迅速的系统。

![实时更新](https://www.milanjovanovic.tech/blogs/mnw_117/async_api_request_with_push.png?imwidth=3840)

SignalR 和 WebSockets 支持服务器与客户端之间的实时通信。当作业状态发生变化时，服务器会立即通知相关客户端。这种方法减少了网络流量，并为用户提供即时反馈。

对于耗时较长的工作，电子邮件通知更有意义。用户无需保持浏览器打开状态。他们可以关闭标签页，收到通知后再回来查看。这对于需要数小时才能生成的报告或夜间运行的批处理过程非常有效。

Webhooks 提供了另一种选择，特别适用于系统间的通信。当作业完成时，你的服务器可以通知其他系统。这实现了工作流程自动化和系统集成，而无需持续轮询。

## 总结

异步处理任务为每个人创造了更好的体验。用户可以立即获得响应，而不必看着旋转的加载指示器。他们可以在等待的同时开始其他任务，并且会知道是否出现了问题。

这些好处不仅仅限于用户体验。服务器可以处理更多请求，因为它们不会被长时间运行的任务所占用。后台处理器可以重试失败的操作，而不会影响主应用程序。你甚至可以将处理能力与 Web 服务器分开进行扩展。

错误处理也得到了改善。当一个长时间操作中途失败时，你可以保存进度并重试。用户确切地知道发生了什么，因为他们可以检查状态。系统保持稳定，因为一个缓慢的操作不会导致整个 API 崩溃。

今天就到这里。期待与你再次相见。
