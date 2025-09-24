# [.NET 中的 gRPC 客户端负载均衡](https://www.rebin.dev/post/grpc-client-side-load-balancing-indotnet/)

## 什么是 gRPC？

gRPC 是一种现代开源框架技术，能帮助我们更高效、高性能地构建应用程序。这项技术通常用于微服务环境中，在组件相互依赖或相互调用时进行通信，gRPC 使用 HTTP/2 上的协议缓冲区来序列化结构化数据。幸运的是，在.NET 生态系统中，我们可以非常轻松地使用 gRPC。

## 客户端负载均衡

负载均衡(LB)允许我们将网络流量分配到多个后端服务(实例)上，以提高应用程序的性能和可靠性，它可分为两种类型：

- 第 4 层传输层协议(TCP、UDP)负载均衡器。
- 第 7 层应用层协议（HTTP、TLS、DNS）负载均衡器。

它们各自使用不同的算法在后台服务（实例）和客户端应用程序之间分配所有请求，最常见的算法在负载均衡中被广泛使用（轮询、加权轮询、最少连接、最短响应时间等）。在本文中，我们尝试配置第 7 层，即应用层负载均衡器。这就是核心思想！客户端应用程序会从后台服务的 IP 地址列表或 DNS 服务记录列表中发送和接收请求，客户端应用程序尝试从列表中随机选择一个 IP 地址来发送和接收 HTTP 请求。

> 今天，我们将创建一个处理.NET 博客订阅源的 gRPC 服务应用程序，并检索包含标题、摘要、发布日期和链接的最新博客文章。该 gRPC 服务会将订阅源数据解析为缓冲协议，然后由 gRPC 客户端应用程序接收。

这是 gRPC 服务后端。

```csharp
public class MainFeedService : FeedService.FeedServiceBase
{
    private string feedUrl = "https://devblogs.microsoft.com/dotnet/feed/";
    private readonly ILogger<MainFeedService> _logger;

    public MainFeedService(ILogger<MainFeedService> logger)
    {
        _logger = logger;
    }


    public override async Task<FeedResponce> GetFeed(Empty request, ServerCallContext context)
    {
        var httpContext = context.GetHttpContext();
        
        await context.WriteResponseHeadersAsync(new Metadata { { "host", $"{httpContext.Request.Scheme}://{httpContext.Request.Host}" } });
        
        FeedResponce listFeedResponce = new();
        
        using XmlReader reader = XmlReader.Create(feedUrl, new XmlReaderSettings { Async = true });
        
        SyndicationFeed feed = SyndicationFeed.Load(reader);
        
        var query = from feedItem in feed.Items.OrderByDescending(x => x.PublishDate)
            select new FeedModel
            {
                Title = feedItem.Title.Text,
                Summary = feedItem.Summary.Text,
                Link = feedItem.Links[0].Uri.ToString(),
                PublishDate = feedItem.PublishDate.ToString()
            };

        listFeedResponce.FeedItems.AddRange(query.ToList());
        
        reader.Close();
        
        return await Task.FromResult(listFeedResponce);
    }
}
```

在 gRPC 服务中，我们向 MetaData 添加了（host），它表示为键值对，类似于 HTTP 头，host 键保存主机地址值，例如 host = <http://localhost:5555>

```csharp
var httpContext = context.GetHttpContext();
await context.WriteResponseHeadersAsync(new Metadata { { "host", $"{httpContext.Request.Scheme}://{httpContext.Request.Host}" } });
```

gRPC 服务后端和客户端具有以下 .proto 文件

```proto
syntax = "proto3";

option csharp_namespace = "GrpcServiceK8s";
import "google/protobuf/empty.proto";
package mainservice;
 

// 信息流服务定义
service FeedService {
  rpc GetFeed (google.protobuf.Empty) returns (FeedResponce);
}
 
// 响应消息包含信息流项列表
message FeedResponce {
 repeated FeedModel FeedItems = 1;
}


// 响应消息包含信息流项的详细信息
message FeedModel {
 string title = 1;
 string summary = 2;
 string link = 3;
 string publishDate = 4;
}
```

## 配置地址解析器

在 gRPC 客户端要启用负载均衡，需要一种解析地址的方法，幸运的是.NET 提供了多种配置 gRPC 解析器的方式，例如：

- StaticResolverFactory 静态解析器
- Custom Resolver 自定义解析器
- DnsResolverFactory DNS 解析器

## StaticResolverFactory

在静态解析器方法中，解析器不会从外部源调用地址，如果我们已经知道所有服务的地址，这种方法就很适合。

这里是在依赖注入(DI)中注册了 StaticResolverFactory 的 Program.cs 文件。

```csharp
var addressCollection = new StaticResolverFactory(address => new[]
{
    new DnsEndPoint("localhost", 5244),
    new DnsEndPoint("localhost", 5135),
    new DnsEndPoint("localhost", 5155)
});


builder.Services.AddSingleton<ResolverFactory>(addressCollection);

builder.Services.AddSingleton(channelService => {
    var methodConfig = new MethodConfig
    {
        Names = { MethodName.Default },
        RetryPolicy = new RetryPolicy
        {
            MaxAttempts = 5,
            InitialBackoff = TimeSpan.FromSeconds(1),
            MaxBackoff = TimeSpan.FromSeconds(5),
            BackoffMultiplier = 1.5,
            RetryableStatusCodes = { Grpc.Core.StatusCode.Unavailable }
        }
    };
    
    return GrpcChannel.ForAddress("static:///feed-host", new GrpcChannelOptions
    {
        Credentials = ChannelCredentials.Insecure,
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() }, MethodConfigs = { methodConfig } },
        ServiceProvider = channelService
    });
});
```

上面的代码（StaticResolverFactory）返回一个地址集合。

此外，我们还配置了重试策略（MethodConfig）以为 gRPC 客户端应用程序提供更高的可用性，本节是可选的。

因为我们使用了 StaticResolverFactory，所以地址的架构应该是静态的（static:///feed-host）

.NET 中的 gRPC 为我们的项目提供了两种负载均衡策略（Pick First）和（Round Robin），我们配置了 Round Robin 策略（算法）。

配置 Round Robin 策略：

```csharp
ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() }, MethodConfigs = { methodConfig } }
```

配置 Pick First 策略：

```csharp
ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new PickFirstConfig() }, MethodConfigs = { methodConfig } }
```

此策略从解析器获取地址列表，然后尝试连接这些地址，直到找到一个可连接的地址。

## 自定义解析器

有时候我们需要为问题创建自定义解决方案，幸运的是，.NET 允许我们创建自定义解析器，我们可以实现（Resolver 和 ResolverFactory）对象来完成这个目标。例如，我们在本地机器上有一个 XML 文件（addresses.xml），其中包含服务地址列表。

```csharp
public class XmlFileResolver : Resolver
{
    private readonly Uri _uriAddress;

    private Action<ResolverResult>? _listener;


    public XmlFileResolver(Uri uriAddress)
    {
        _uriAddress = uriAddress;
    }


    public override async Task RefreshAsync(CancellationToken cancellationToken)
    {
        await Task.Run(async () =>
        {
            using (var reader = new StreamReader(_uriAddress.LocalPath))
            {
                XmlSerializer serializer = new XmlSerializer(typeof(Root));

                var resultsDeserialize = (Root)serializer.Deserialize(reader);

                IReadOnlyList<DnsEndPoint> listAddresses = resultsDeserialize.Element.Select(r => new DnsEndPoint(r.HostName, r.Port)).ToList();

                _listener(ResolverResult.ForResult(listAddresses, serviceConfig: null));
            }
        });
    }


    public override void Start(Action<ResolverResult> listener)
    {
        _listener = listener;
    }
}

public class CustomFileResolver : ResolverFactory
{
    public override string Name => "file";

    public override Resolver Create(ResolverOptions options)
    {
        return new xmlFileResolver(options.Address);
    }
}
```

自定义解析器创建后，需要在依赖注入(DI)中注册，同时还需要提供文件路径，如下面的代码所示。

```csharp
builder.Services.AddSingleton<ResolverFactory, customFileResolver>();

builder.Services.AddSingleton(channelService =>
{
    return GrpcChannel.ForAddress("file:///c:/urls/addresses.xml", new GrpcChannelOptions
    {
        Credentials = ChannelCredentials.Insecure,
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() } },
        ServiceProvider = channelService
    });
});
```

## DnsResolverFactory

在这种方法中，DNS 解析器将依赖外部源获取包含后端服务（实例）地址和端口的服务记录，为了配置负载均衡，我们使用 Kubernetes 中的无头服务，它为 gRPC 客户端的 DnsResolver 提供了所有必要条件，为了更好地理解这个概念，我们有以下交互式图表。

![DNS](https://www.rebin.dev/images/Kubernetes-HeadlessService.png)

首先，我们需要在 Kubernetes 中创建 gRPC 服务后端 pod。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service-pod
  labels:
    app: grpc-service-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: grpc-service-app
  template:
    metadata:
      labels:
        app: grpc-service-app
    spec:
      containers:
      - name: grpcservice
        image: rebinoq/grpcservicek8s:v1
        resources:
            requests:
              memory: "64Mi"
              cpu: "125m"
            limits:
              memory: "128Mi" 
              cpu: "250m"
        ports:
        - containerPort: 80
```

运行以下命令在 Kubernetes 中创建服务 Pod。

```bash
kubectl create -f grpc-service-pod.yaml
```

要在 Kubernetes 中创建 Headless 服务，我们运行以下 yaml 代码。

注意：在 Kubernetes 环境中创建无头服务(svc)时，clusterIP 键必须设置为'none'值。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-headless-svc
spec:
  clusterIP: None 
  selector:
    app: grpc-service-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

运行以下命令来创建 Headless 服务。

```bash
kubectl create -f grpc-headless-svc.yaml
```

是时候检查 Headless 服务和 DNS 解析器的状态了，我们运行以下命令。

```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml

kubectl exec -i -t dnsutils -- nslookup grpc-headless-svc       
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   grpc-headless-svc.default.svc.cluster.local
Address: 10.1.1.163
Name:   grpc-headless-svc.default.svc.cluster.local
Address: 10.1.1.165
Name:   grpc-headless-svc.default.svc.cluster.local
Address: 10.1.1.167
```

到目前为止一切顺利，我们终于获得了服务（grpc-headless-svc.default.svc.cluster.local），其中包含了三个 Pod 的 IP 地址

我们再次为 gRPC 客户端创建另一个 Pod，只需运行以下 yaml 代码。

注意：我们在环境变量中向 gRPC 客户端传递了 dns:///grpc-headless-svc。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-client-pod
  labels:
    app: grpc-client-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grpc-client-app
  template:
    metadata:
      labels:
        app: grpc-client-app
    spec:
      containers:
      - name: grpcclient
        image: rebinoq/grpcclientk8s:v1
        resources:
            requests:
              memory: "64Mi"
              cpu: "125m"
            limits:
              memory: "128Mi" 
              cpu: "250m"
        ports:
        - containerPort: 80
        env:
        - name: k8s-svc-url
          value: dns:///grpc-headless-svc
```

运行命令

```bash
kubectl create -f grpc-client-pod.yaml
```

我们在 Kubernetes 中需要创建的最后一样东西叫做 Service Node，这个服务使我们能够从 Kubernetes 外部访问 gRPC 客户端 Pod。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-nodeport-svc
spec:
  type: NodePort
  selector:
    app: grpc-client-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30030
```

运行命令

```bash
kubectl create -f grpc-nodeport-svc.yaml
```

由于这个端口（30030），我们可以访问 gRPC 客户端应用程序。例如 <http://localhost:30030>

到目前为止，所有操作都是在 Kubernetes 上完成的，现在让我们看看 gRPC 客户端中的 DnsResolver 配置代码。

这是 DNS 解析器代码

```csharp
builder.Services.AddSingleton<ResolverFactory>(new DnsResolverFactory(refreshInterval: TimeSpan.FromSeconds(25)));

builder.Services.AddSingleton(channelService =>
{
    var methodConfig = new MethodConfig
    {
        Names = { MethodName.Default },
        RetryPolicy = new RetryPolicy
        {
            MaxAttempts = 5,
            InitialBackoff = TimeSpan.FromSeconds(1),
            MaxBackoff = TimeSpan.FromSeconds(5),
            BackoffMultiplier = 1.5,
            RetryableStatusCodes = { Grpc.Core.StatusCode.Unavailable }
        }
    };

    var configuration = builder.Configuration;

    return GrpcChannel.ForAddress(configuration.GetValue<string>("k8s-svc-url"), new GrpcChannelOptions
    {
        Credentials = ChannelCredentials.Insecure,
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() }, MethodConfigs = { methodConfig } },
        ServiceProvider = channelService
    });
});
```

每当任何连接断开时，DNS 解析器将刷新并立即尝试检测 gRPC 服务的另一个 Pod（实例）。

```csharp
builder.Services.AddSingleton<ResolverFactory>(new DnsResolverFactory(refreshInterval: TimeSpan.FromSeconds(25)));
```

这是 gRPC 客户端代码。

```csharp
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;
    private readonly GrpcChannel _grpcChannel;
    private List<FeedViewModel>? _feedViewModel;

    public HomeController(ILogger<HomeController> logger, GrpcChannel grpcChannel)
    {
        _logger = logger;
        _grpcChannel = grpcChannel;

    }

    public async Task<IActionResult> Index()
    {
        var client = new FeedService.FeedServiceClient(_grpcChannel);
        
        try
        {
            var replay = client.GetFeedAsync(new Google.Protobuf.WellKnownTypes.Empty());

            var responseHeaders = await replay.ResponseHeadersAsync;

            ViewData["hostAdress"] = responseHeaders.GetValue("host");

            var responseFeed = await replay;

            _feedViewModel = new List<FeedViewModel>();

            foreach (var item in responseFeed.FeedItems)
            {
                _feedViewModel.Add(new FeedViewModel
                {
                    title = item.Title,
                    Summary = item.Summary,
                    publishDate = DateTime.Parse(item.PublishDate),
                    Link = item.Link
                });
            }
        }
        catch (RpcException rpcex)
        {
            _logger.LogWarning(rpcex.StatusCode.ToString());
            _logger.LogError(rpcex.Message);
        }

        return View(_feedViewModel);
    }
}
```

gRPC 客户端负载均衡的最终结果。

![运行结果](https://www.rebin.dev/images/LoadBlancing.gif)

今天就到这里。期待与你再次相见。
