# [使用 gRPC 流媒体的秘诀 - 如何将它应用到实时 LLM 信息流](https://medium.com/c-sharp-programming/2025s-most-stolen-grpc-streaming-secret-and-how-to-hijack-it-for-real-time-llm-feeds-17ff014efadc)

我们有一个连接到本地 LLM 实例的 WebSocket 管道，通过 SignalR 流式传输到一个重度使用 React 的控制台。我们的任务是什么？逐个令牌传递，以低于 16 毫秒的延迟将 AI 输出流式传输到浏览器，完美适配 60fps 渲染。

理论上，这看起来像是 GPU 瓶颈。

实际上？问题隐藏在 gRPC、网络缓冲区和浏览器绘制队列之间的某个地方。

在那个冲刺阶段结束时，我们已经两次重写了整个流堆栈。构建了一个自定义的 HTTP/3 传输层。几乎用 FlatBuffers 替换了 Protobuf。然后出现了那个模式——那个反模式——改变了一切。

## "标准"gRPC 流式设置（为何会出问题）

大多数人在他们的 .proto 文件中是这样开始的：

```proto
service LLMStreamer {
  rpc StreamTokens(TokenRequest) returns (stream TokenResponse);
}

message TokenRequest {
  string prompt = 1;
  int32 max_tokens = 2;
}
message TokenResponse {
  string token = 1;
  float probability = 2;
}
```

它编译得很完美。

你的.NET 服务器可能看起来像这样：

```csharp
public override async Task StreamTokens(TokenRequest request, IServerStreamWriter<TokenResponse> responseStream, ServerCallContext context)
{
    var generator = _llm.GenerateAsync(request.Prompt, request.MaxTokens);
    await foreach (var token in generator.WithCancellation(context.CancellationToken))
    {
        await responseStream.WriteAsync(new TokenResponse
        {
            Token = token.Text,
            Probability = token.Confidence
        });
    }
}
```

看起来很干净。但很快就会崩溃。

令牌到达了……但并不一致。有时你会一次性收到 3 个令牌。有时 200 毫秒内什么都没有。在本地循环中，你不会注意到。在真实网络上呢？实时性的幻觉就崩溃了。

## 秘诀：调整流节奏，而不是追逐它

突破来自一个不相关的领域——游戏引擎。它不是 gRPC。不是 AI。而是多人竞技场中的网络帧同步。

我们意识到不需要在令牌准备就绪时立即发送它们。我们需要每 16 毫秒刷新一次。帧锁定。可预测。与浏览器同步。

这是我们重新设计的服务器端处理程序的样子：

```csharp
public override async Task StreamTokens(TokenRequest request, IServerStreamWriter<TokenResponse> responseStream, ServerCallContext context)
{
    var tokenBuffer = new Channel<TokenResponse>(capacity: 100);
    
    _ = Task.Run(async () =>
    {
        await foreach (var token in _llm.GenerateAsync(request.Prompt, request.MaxTokens)
                                        .WithCancellation(context.CancellationToken))
        {
            await tokenBuffer.Writer.WriteAsync(new TokenResponse
            {
                Token = token.Text,
                Probability = token.Confidence
            });
        }
        tokenBuffer.Writer.Complete();
    });
    var flushInterval = TimeSpan.FromMilliseconds(16); // ~60fps
    var tokensToSend = new List<TokenResponse>();
    var nextFlush = Stopwatch.StartNew();
    await foreach (var token in tokenBuffer.Reader.ReadAllAsync(context.CancellationToken))
    {
        tokensToSend.Add(token);
        if (nextFlush.Elapsed >= flushInterval)
        {
            foreach (var t in tokensToSend)
                await responseStream.WriteAsync(t);
            tokensToSend.Clear();
            nextFlush.Restart();
        }
    }
    
    foreach (var t in tokensToSend)
        await responseStream.WriteAsync(t);
}
```

我们不是在延迟令牌。我们是在编排它们。

突然间，用户界面感觉更快了——即使总流式传输时间保持不变。

## 前端：60 帧或失败

服务器节流还不够。我们需要教会浏览器尊重帧预算。这意味着使用 requestAnimationFrame 并让浏览器决定何时绘制。

```javascript
const tokens: string[] = [];
let renderBuffer: string[] = [];

const client = new LLMServiceClient("https://api.example.com");
const stream = client.streamTokens({ prompt: inputPrompt, maxTokens: 256 });
stream.on("data", (response) => {
  tokens.push(response.token);
});
function renderLoop() {
  if (tokens.length > 0) {
    renderBuffer.push(tokens.shift()!);
    updateDOM(renderBuffer.join(""));
  }
  requestAnimationFrame(renderLoop);
}
renderLoop();
```

在底层， updateDOM() 在 React 之上使用了一个差异层来减少 DOM 操作。但真正的技巧在于节流——始终按照屏幕的节奏，绝不更快。

没有绘制抖动。没有布局颠簸。没有帧丢失。

## 这种模式被迅速采用

在我们发布概念验证的一周内，开发者们就在 Reddit、GitHub，甚至大型 AI 实验室的内部 Slack 频道上讨论了它。

每种变体都添加了自己的特色。预测性 token 预览。多流融合。自适应延迟控制。

但有一个部分保持不变：节奏控制层。

那 16 毫秒的脉冲就是"相当快"和"感觉实时"之间的区别。

## 最终思考

Protobuf 很快。gRPC 很强大。LLMs 正在改变一切。

但性能不仅仅是数学问题。它是心理学问题。

最重要的是令牌何时到达屏幕——而不是它离开模型的速度有多快。

这就是秘诀。现在它属于你了。
