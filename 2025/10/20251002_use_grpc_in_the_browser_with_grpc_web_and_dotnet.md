# [如何在 .NET 中使用 gRPC-Web 让浏览器直接调用 gRPC 服务](https://medium.com/geekculture/build-high-performance-services-with-grpc-and-net-5-part-2-92c4561b8ac2)

今天，我们将探讨微软的 gRPC-Web 以及如何使用它来创建可以从浏览器调用的实际 gRPC 服务。

## 面临的挑战

在之前的讨论中，我们已经看到，与 REST 不同，gRPC 服务无法直接从浏览器调用。

当从浏览器访问时，gRPC 项目模板被配置为显示以下警告：

> Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: <https://go.microsoft.com/fwlink/?linkid=2086909>

这是因为 gRPC 依赖于 HTTP/2 尾部响应头（也称为 trailers），而浏览器 API 不支持这些头部。

那么让我们通过概述 HTTP/2 来了解那些尾随响应头。

![HTTP 1.1 和 HTTP/2 高级结构](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dgNgwMVW6M3WEWgdXC3sGg.jpeg)

从上图可以看出，HTTP/2 有一个帧机制——这是 HTTP/1.1 和底层传输协议之间的一个中间层。

HTTP/2 消息被分割成帧，然后嵌入到流中。将数据帧和头帧分离为进一步优化（头压缩和多路复用）打开了大门。

使用 HTTP/2，服务器可以针对一个请求发送多个响应，因此默认的 HTTP 状态 200/OK 将不够用。

为了解决这个问题——使用尾部标头来获取有关传入响应的元数据。例如，gRPC-status 可以通知客户端流中的消息是否已成功接收。

## 解决方案

这就是 gRPC-Web 的用武之地——它位于浏览器和 gRPC 服务器之间，充当一个同时兼容 HTTP 1.1 和 HTTP 2 的代理。

![gRPC-Web](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*kPjX1qriawYbyrynd9-6WQ.png)

这项技术并不新鲜，它基于 grpc-web JavaScript 客户端。

## .NET gRPC-Web 入门指南

### 配置你的 gRPC 服务项目

使用"gRPC Service"项目模板添加一个新的 gRPC ASP.NET Core 服务。

让我们不使用默认的 greet.proto，而是添加另一个 protobuf 文件（stock.proto），如下所示：

```proto
syntax = "proto3";

option csharp_namespace = "StockServices.Protos";

package stock;
 
service Stock {
  rpc GetAllProducts (Empty) returns (stream Product);
  rpc AddProduct (Product) returns (Result);
}
  
message Empty {}

message Result {
  bool status = 1;
  string msg = 2;
}

message Product {
  string name = 1;
  uint32 code = 2;
  uint32 stock = 3;
}
```

这个 proto 文件包含一个服务，该服务有两个方法

1. AddProduct — 它通过一元调用添加产品。
2. GetAllProducts — 它返回一个产品流。要启用流式传输，我们需要添加 stream 前缀。

然后我们有针对 Product 和 Result 的自定义消息。

提示：当你在解决方案中添加任何 protobuf 文件时。请确保将"生成操作"设置为"protobuf 编译器"，如下所示：

![将生成操作设置为 Protobuf 编译器](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*QQYQmmutjyyQ5Gi95lTCMQ.png)

最后在 Services 文件夹中添加一个新类 — StockService.cs。

然后你可以从 {service-name}.{service-name}.Base 继承你的类

```csharp
using StockServices.Protos;

public class StockService : Stock.StockBase {

}
```

提示：grpc 代码生成会自动添加 .proto 命名空间，所以你需要添加它的引用，例如 — {your-name-space}.Protos

在我们的 StockService 类中定义一个静态产品存储。这将保存所有由客户端添加的产品对象。

```csharp
private static List<Product> _allProducts = new List<Product>();
```

通过输入 override 来重写存根类方法。你将看到在你的 proto 文件中定义的所有方法。

```csharp
public override Task<Result> AddProduct(Product request, ServerCallContext context)
{
}
```

我们将为此方法添加以下功能，以便将产品添加到我们的内存产品存储中。

```csharp
if (string.IsNullOrEmpty(request.Name))
    return Task.FromResult<Result>(new Result { Msg = "Product Name Can't be nulled", Status = false });
if (_allProducts.FirstOrDefault(f => f.Code == request.Code) != null)
    return Task.FromResult<Result>(new Result { Msg = "Product is already Added", Status = false });

_allProducts.Add(request);

return Task.FromResult<Result>(new Result { Msg = "Added. Total Products: "+_allProducts.Count.ToString(), Status = true });
```

让我们也实现我们的流式传输方法，该方法将简单地将产品存储中添加的所有产品流式传输到客户端。

```csharp
public override async Task GetAllProducts(Empty request, IServerStreamWriter<Product> responseStream, ServerCallContext context)
{
    foreach (var each in _allProducts) {
        await responseStream.WriteAsync(each);
    } 
}
```

现在，为了使此服务可以从浏览器调用，我们必须执行以下步骤：

1. 安装 Grpc.AspNetCore 包
2. 在 Startup.cs 的 Configure 方法中添加以下代码行。

```csharp
app.UseGrpcWeb();
app.UseEndpoints(endpoints =>
{
    endpoints.MapGrpcService<StockService>().EnableGrpcWeb();
});
```

这个中间件将处理所有关于 grpc-web 调用的事情，我们的 gRPC 服务不需要了解任何关于 grpc-web 的信息。

就这样 — 你的 gRPC 服务现在可以从浏览器中调用了。

### 使用（Blazor、SPA 或 Javascript）创建你的 Web 客户端

让我们添加一些 HTML 和 JavaScript 客户端代码来与此服务交互。

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>gRPC-Web | Demo</title>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta/css/bootstrap.min.css" integrity="sha384-/Y6pD6FV/Vv2HJnA6t+vslU6fwYXjCFtcEpHbNJ0lyAFsXTsjBbfaDjzALeQsN6M" crossorigin="anonymous">
    
</head>
<body>
    <div class="container">
        <h3>gRPC-Web | Demo</h3>
        <br />
        <div>
            <div class="form-group">
                <label for="name">Product Name</label>
                <input type="text" class="form-control" id="name" placeholder="Enter product name ">
            </div>

            <div class="form-group">
                <label for="age">Product Code</label>
                <input type="number" class="form-control" id="code" placeholder="enter product code here " min="0" max="100" step="1" value="0">
            </div>

            <button type="button" id="addProduct" class="btn btn-primary">Add Product</button>


            <button type="button" id="getAllProducts" class="btn btn-primary">Get All Products </button>
          
            <div>
                <p id="result"></p>
            </div>
        </div>
        <div class="container">

            <table class="table">
                <thead>
                    <tr>
                        <th>Product Name</th>
                        <th>Product Code</th>
                    </tr>
                </thead>
                <tbody id="trProducts">
                    <tr>
                    </tr>

                </tbody>
            </table>
        </div>
    </div>
    <script type="text/javascript" src="dist/main.js"></script>
</body>
</html>
```

这个 HTML 文件接受两个输入（产品名称和代码），并调用我们 gRPC 服务中的 AddProduct 和 getAllProducts 方法。

![页面展示](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HYzQHzcflvDONveBQkNf4A.png)

现在我们可以在 wwwroot 中添加"Scripts"文件夹来存放我们的脚本文件，例如 index.js。

```javascript
const { Product, Empty, Result } = require('./stock_pb');
const { StockClient } = require('./stock_grpc_web_pb');

var client = new StockClient(window.location.origin);

var txtName = document.getElementById('name');
var txtCode = document.getElementById('code');
var btnAddProduct = document.getElementById('addProduct');
var trProducts = document.getElementById('trProducts');

var getAllProducts = document.getElementById('getAllProducts');
var resultText = document.getElementById('result');


getAllProducts.onclick = function () {
    trProducts.innerHTML = "";
    getProductsStream();

};

 
btnAddProduct.onclick = function () {
    var request = new Product();
    request.setName(txtName.value);
    request.setCode(txtCode.value);

    client.addProduct(request, {}, (err, response) => {
           resultText.innerHTML = htmlEscape(response.getMsg());
    });
};

 
function getProductsStream () {
    var request = new Empty();
    streamingCall = client.getAllProducts(request, {});
    streamingCall.on('data', function (response) {
     trProducts.innerHTML += "<tr><td>" + htmlEscape(response.getName()) + "</td><td>" + htmlEscape(response.getCode())+"</td></tr>";
    });

    streamingCall.on('end', function () {
        console.log("Stream ended");
     });
   
};
 
function htmlEscape(str) {
    return String(str)
        .replace(/&/g, '&amp;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#39;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;');
}
```

你可以注意到它需要'stock_pb'和'stock_grpc_web_pb'文件。这些是 gRPC-Web 的 JavaScript 客户端和消息，它们是使用'protoc'（gRPC-Web 代码生成器插件）生成的。

你可以使用以下命令在脚本文件夹中生成这些文件。

```bash
protoc greet.proto --js_out=import_style=commonjs:CHANGE_TO_SCRIPTS_DIRECTORY --grpc-web_out=import_style=commonjs,mode=grpcwebtext:CHANGE_TO_SCRIPTS_DIRECTORY --plugin=protoc-gen-grpc-web=CHANGE_TO_PROTOC_GEN_GRPC_WEB_EXE_PATH
```

注意：确保你的计算机上已安装'protoc'和'Protoc-gen-grpc-web'，并且可以从 PATH 中找到它们。

让我们分析一下我们的 JavaScript 代码：

这将创建我们的 gRPC 客户端——我们将使用这个对象来调用服务方法。

```javascript
var client = new StockClient(window.location.origin);
```

消息可以按如下方式初始化：

```javascript
var request = new Product(); 
request.setName(txtName.value); 
request.setCode(txtCode.value);
```

最后，我们可以按如下方式调用 gRPC 服务：

```javascript
client.addProduct(request, {}, (err, response) => { 
    resultText.innerHTML = htmlEscape(response.getMsg()); 
});
```

以下是我们的 package.json 文件，由于我们正在使用 webpack 来

```json
{
  "version": "1.0.0",
  "name": "browser-server",
  "private": true,
  "devDependencies": {
    "@grpc/proto-loader": "^0.3.0",
    "google-protobuf": "^3.6.1",
    "grpc": "^1.15.0",
    "grpc-web": "^1.0.0",
    "webpack": "^4.16.5",
    "webpack-cli": "^3.1.0"
  }
}
```

注意：此示例需要 node.js 和 webpack，因此你需要更新构建目标。

让我们构建并运行项目。

尝试添加一个新产品——你将看到你的 content-type 为'application/grpc-web-text'的 gRPC 调用：

![gRPC-Web 调用](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TaNKCAYXBz65-qGfdYIirA.png)

与纯文本 JSON 不同，响应是 base64 编码的（包含二进制字节）。

### 服务器流式传输

我们的演示应用对"添加产品"功能使用一元调用，对"获取所有产品"功能使用流式调用。你可以尝试添加几个产品，然后点击"获取所有产品"。这将导致加载所有已添加的产品。

![页面展示](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dIrMNZm0kAubnWqQDAfC-w.png)

这是通过服务器流式传输完成的——客户端发送一个 gRPC 调用，并获得一个流来读取返回的消息序列。

```javascript
var request = new Empty(); 
streamingCall = client.getAllProducts(request, {}); 
streamingCall.on('data', function (response) { 
    trProducts.innerHTML += "<tr><td>" + htmlEscape(response.getName()) + "</td><td>" + htmlEscape(response.getCode())+"</td></tr>"; 
});
```

如果你想知道流何时结束——你可以订阅这个函数：

```javascript
streamingCall.on('end', function () {
    console.log("Stream ended");
});
```

## 总结

我希望这篇文章能让你很好地了解如何使用 gRPC-Web 与 gRPC 服务进行通信。在性能至关重要且你需要利用多路复用或服务器流式传输的强大功能时，你可能会选择 gRPC-Web 而非 REST/JSON。

今天就到这里。期待与你再次相见。
