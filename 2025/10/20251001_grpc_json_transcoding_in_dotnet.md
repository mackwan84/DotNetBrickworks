# [.NET 中的 gRPC JSON 转码](https://poornimanayar.co.uk/blog/grpc-json-transcoding-in-net-7/)

> gRPC JSON 转码是 .NET 中引入的新功能之一。让我们通过本文来了解更多关于这个功能的信息。

gRPC 是构建 API 的绝佳框架。这是谷歌对远程过程调用(RPC)的实现，并且在.NET 中获得了官方支持。gRPC.NET 现在是.NET 世界中远程过程调用需求的推荐框架。gRPC 得益于出色的性能，并且是为 HTTP/2 及更高版本设计的。但是，如果你希望有基于浏览器的客户端来使用你的 API，它并不是一个好的选择。要使用 gRPC API，你需要一个生成的 gRPC 客户端，而浏览器无法提供支持 gRPC 客户端所需的网络请求控制级别。但如果你有一个基于浏览器的应用程序想要使用这个 gRPC 服务怎么办？你可以将 gRPC 服务使用的代码重用/复制到 HTTP API 中，但这意味着大量代码重复和维护工作。这正是 gRPC JSON 转码可以派上用场的场景。它降低了 gRPC API 的入门门槛，使基于浏览器的应用程序可以使用它，而无需代码重复。你可以使用此功能将任何现有的 gRPC API 扩展为 HTTP API。

使用 gRPC 服务时，客户端应用程序通过 HTTP/2 POST 与服务通信。请求和响应使用 Protobuf 格式，并利用 Google.Protobuf 进行序列化和反序列化。最重要的是，客户端必须使用生成的客户端来与服务通信。

![gRPC 通信](https://poornimanayar.co.uk/media/4wjjytij/image.png)

通过 gRPC JSON 转码，任何能够发出 HttpRequest 的应用程序都可以使用该服务。应用程序可以使用一种众所周知的 HTTP 方法发出 HTTP/1.1 或 HTTP/2 请求。JSON 被用作消息交换格式，并使用内置的 System.Text.Json 库进行序列化和反序列化。

![gRPC JSON 转码通信](https://poornimanayar.co.uk/media/yiafwghc/image.png)

通过这种方法，gRPC 服务可以继续正常运行，同时也可以作为 HTTP API 使用。无需重复代码。你可以重用 gRPC 服务中已有的代码，但 API 被扩展为也可以作为 HTTP API 使用，从而使你的 API 覆盖范围更广。此外，这是一个单一的 ASP.NET Core 应用程序，同时作为 gRPC 和 HTTP API 运行，使你的代码更易于维护，托管也更加方便。

现在让我们来看看实际操作。我们首先需要一个 gRPC 服务，然后我们将扩展它以实现 gRPC JSON 转码。

我将从 .proto 文件开始，该文件如下所示。我的文件名为 festiveapi.proto 。它包含一个名为 FestiveAdventService 的服务定义。它有多个 rpc 方法 - 用于列出消息、获取消息、创建消息、更新消息和删除消息。

```proto
syntax = "proto3";

option csharp_namespace = "FestiveAdventApi";

package festiveadvent;

import "google/protobuf/empty.proto";

service FestiveAdventService {
  rpc ListMessages(google.protobuf.Empty) returns (ListReply);
  rpc GetMessage(GetAdventMessageRequest) returns (AdventMessageReply);
  rpc CreateMessage(CreateAdventMessageRequest) returns (AdventMessageReply);
  rpc UpdateMessage(UpdateAdventMessageRequest) returns (AdventMessageReply);
  rpc DeleteMessage(DeleteAdventMessageRequest) returns (google.protobuf.Empty);
}

message CreateAdventMessageRequest{
  int32 id = 1;
  string message = 2;
}

message GetAdventMessageRequest{
  int32 id = 1;
}

message UpdateAdventMessageRequest{
  int32 id = 1;
  string message = 2;
}

message DeleteAdventMessageRequest{
  int32 id = 1;
}

message ListReply{
  repeated AdventMessageReply AdventMessages = 1;
}

message AdventMessageReply{
  int32 id = 1;
  string message = 2;
}
```

我选择了 Visual Studio 开箱即用的 Grpc 服务模板作为我的起点。它已经包含了所有必要的 Nuget 包（ Grpc.AspNetCore ）。当我使用上述 proto 文件构建项目时，工具会自动启动并为我生成代码。现在我可以为 rpc 方法提供我自己的实现。

```csharp
using FestiveAdvent.Data;
using FestiveAdvent.Repository;
using FestiveAdventApi;
using Google.Protobuf.WellKnownTypes;
using Grpc.Core;

namespace FestiveAdvent.Services;

public class FestiveAdventApiService : FestiveAdventService.FestiveAdventServiceBase
{
    private readonly FestiveAdventRepository _festiveAdventRepository;


    public FestiveAdventApiService(FestiveAdventRepository festiveAdventRepository)
    {
        _festiveAdventRepository = festiveAdventRepository;
    }

    public override Task<ListReply> ListMessages(Empty request, ServerCallContext context)
    {
        var items = _festiveAdventRepository.GetAdventMessages();

        var listReply = new ListReply();
        var messageList = items.Select(item => new AdventMessageReply() { Id = item.Id,Message = item.Message }).ToList();
        listReply.AdventMessages.AddRange(messageList);
        return Task.FromResult(listReply);
    }

    public override Task<AdventMessageReply> GetMessage(GetAdventMessageRequest request, ServerCallContext context)
    {
        var message = _festiveAdventRepository.GetAdventMessage(request.Id);
        return Task.FromResult(new AdventMessageReply() { Id = message.Id, Message = message.Message });
    }

    public override Task<AdventMessageReply> CreateMessage(CreateAdventMessageRequest request, ServerCallContext context)
    {
        var messageToCreate = new AdventMessage()
        {
            Id=request.Id,
            Message = request.Message
        };

        var createdMessage = _festiveAdventRepository.AddAdventMessage(messageToCreate);

        var reply = new AdventMessageReply() { Id = createdMessage.Id, Message = createdMessage.Message};

        return Task.FromResult(reply);
    }

    public override Task<AdventMessageReply> UpdateMessage(UpdateAdventMessageRequest request, ServerCallContext context)
    {
        var existingMessage = _festiveAdventRepository.GetAdventMessage(request.Id);

        if (existingMessage == null)
        {
            throw new RpcException(new Status(StatusCode.NotFound, "Advent message not found"));
        }

        var messageToUpdate = new AdventMessage()
        {
            Id=request.Id,
            Message = request.Message
        };

        var createdMessage = _festiveAdventRepository.UpdateAdventMessages(messageToUpdate);

        var reply = new AdventMessageReply() { Id = createdMessage.Id, Message = createdMessage.Message};

        return Task.FromResult(reply);
    }

    public override Task<Empty> DeleteMessage(DeleteAdventMessageRequest request, ServerCallContext context)
    {
        var existingMessage = _festiveAdventRepository.GetAdventMessage(request.Id);

        if (existingMessage == null)
        {
            throw new RpcException(new Status(StatusCode.NotFound, "Advent message not found"));
        }

        _festiveAdventRepository.DeleteAdventMessage(request.Id);

        return Task.FromResult(new Empty());
    }
}
```

最后，我必须使用中间件将我的类映射为 gRPC 服务。

```csharp
app.MapGrpcService<FestiveAdventApiService>();
```

我的 gRPC 服务已经启动并运行。现在让我们将这个 gRPC API 也扩展为 HTTP API。

gRPC JSON 转码是 ASP.NET Core 的扩展功能。通过 gRPC JSON 转码，JSON 格式的请求消息会被反序列化为 Protobuf 格式，然后直接调用 gRPC 服务。这里需要理解的最重要概念是 HTTP 规则。

## 理解 HTTP 规则的基础

HTTP 规则定义了 gRPC-HTTP API 映射的模式。每个注解指定了一个路由和该端点的 HTTP 方法。映射定义了 HTTP 方法和路由模板（称为模式）。映射还决定了路由参数、查询字符串和请求体如何映射到 gRPC 请求消息。映射还决定了 gRPC 响应消息如何映射到响应体。HTTP 规则使用注解 google.api.http. 指定

一个典型的 HTTP 规则可能看起来像这样。

```proto
service FestiveAdventService {
  rpc ListMessages(google.protobuf.Empty) returns (ListReply) {
    option (google.api.http) ={
      get : "/message"
    };
  }
}
```

在此示例中，通过添加注解，该 RPC 可以作为 HTTP API 端点调用，并响应 <api_base_url>/message 处的 HTTP GET 请求。它也可以继续作为 gRPC 服务被调用。

在路由模板中也可以包含路由参数。例如，通过 ID 获取消息的 RPC 可以如下所示进行注解。

```proto
service FestiveAdventService {
  rpc GetMessage(GetAdventMessageRequest) returns (AdventMessageReply) {
    option (google.api.http) = {
      get : "/message/{id}"
    };
  }
}

message GetAdventMessageRequest {
  int32 id = 1;
}

message AdventMessageReply {
  int32 id = 1;
  string message = 2;
}
```

在此示例中， GetAdventMessageRequest 消息中的 id 字段被映射到路由参数 id 。

你也可以在路由中传入查询参数。这些参数会被绑定到请求消息中同名的相应字段。

你还可以指定将请求消息绑定到请求正文。在此示例中，要创建消息，请求消息将完全从 JSON 请求正文中绑定。

```proto
service FestiveAdventService {
  rpc CreateMessage(AdventMessageRequest) returns (AdventMessageReply){
    option (google.api.http) = {
      post : "/message"
      body : "*"
    };
  }
}

message AdventMessageRequest {
  int32 id = 1;
  string message = 2;
}
 
message AdventMessageReply {
  int32 id = 1;
  string message = 2;
}
```

约定" * "指定任何未被路由模板绑定的字段都将从请求体中进行绑定。

我在这里仅展示了 HTTP GET 和 POST 的示例，以便涵盖 Protobuf 字段的绑定，但 HTTP 规则支持 GET、POST、PUT、PATCH 和 DELETE（如下面的示例代码所示）。如果你希望实现像 HEAD 这样的 HTTP 方法，可以使用 custom 模式（代替 HTTP 方法）。

现在让我们运用对 HTTP 规则的了解，并将其付诸实践。

首先，我需要安装 Microsoft.AspNetCore.Grpc.JsonTranscoding Nuget 包。

```bash
dotnet add package Microsoft.AspNetCore.Grpc.JsonTranscoding
```

我还需要在 Startup 中注册 JsonTranscoding。

```csharp
builder.Services.AddGrpc().AddJsonTranscoding();
```

最后，附上我的带注释的 proto 文件。

```proto
syntax = "proto3";

option csharp_namespace = "FestiveAdventApi";

package festiveadvent;

import "google/protobuf/empty.proto";
import "google/api/annotations.proto";

service FestiveAdventService {
  rpc ListMessages(google.protobuf.Empty) returns (ListReply) {
    option (google.api.http) = {
      get : "/message"
    };
  }

  rpc GetMessage(CreateAdventMessageRequest) returns (AdventMessageReply) {
    option (google.api.http) = {
      get : "/message/{id}"
    };
  }

  rpc CreateMessage(AdventMessageRequest) returns (AdventMessageReply) {
    option (google.api.http) = {
      post : "/message"
      body : "*"
    };
  }

  rpc UpdateMessage(UpdateAdventMessageRequest) returns (AdventMessageReply){
    option (google.api.http) = {
      put : "/message/{id}"
      body : "*"
    };
  }

  rpc DeleteMessage(DeleteAdventMessageRequest) returns (google.protobuf.Empty){
    option (google.api.http) = {
      delete : "/message/{id}"
    };
  }
}

message CreateAdventMessageRequest {
  int32 id = 1;
  string message = 2;
}

message GetAdventMessageRequest {
  int32 id = 1;
}

message UpdateAdventMessageRequest {
  int32 id = 1;
  string message = 2;
}

message DeleteAdventMessageRequest {
  int32 id = 1;
}

message ListReply {
  repeated AdventMessageReply AdventMessages = 1;
}

message AdventMessageReply {
  int32 id = 1;
  string message = 2;
}
```

我的每个 RPC 都带有注释。

- ListMessages rpc 被映射到 HTTP GET /message 端点
- GetMessage rpc 被映射到 HTTP GET /message/{id} 端点。 GetAdventMessageRequest 消息中的 id 字段从路由参数 id 绑定
- CreateMessage rpc 被映射到 HTTP POST /message 端点。 CreateAdventMessageRequest 消息按照 body : "*" 的指定从请求体绑定。请求消息中的字段被绑定到请求体中同名的相应字段
- UpdateMessage rpc 被映射到 HTTP PUT /message/{id} 端点。 UpdateAdventMessageRequest 消息中的 id 字段被绑定到路由参数 id 。请求消息中的任何其他字段按照 body : "*" 的指定从请求体绑定
- DeleteMessage rpc 被映射到 HTTP DELETE /message/{id} 端点。 DeleteAdventMessageRequest 消息中的 id 字段被绑定到路由参数 id 。

就是这样！！我现在可以运行我的项目并使用 Postman 测试我通过 gRPC JSON 转码创建的 HTTP API。我的 gRPC API 没有任何变化，它可以继续为我现有的 gRPC 客户端提供服务。但我通过一些注解扩展了这个 API，使我的基于浏览器的应用程序能够使用它。正如你所看到的，没有代码重复，它是一个单一的 ASP.NET Core 服务！

## 但这还不是全部

我们也可以为我们的 gRPC JSON 转码 API 提供 OpenAPI 支持。要启动并运行此功能，我们需要安装 Microsoft.AspNetCore.Grpc.Swagger 包

```bash
dotnet add package Microsoft.AspNetCore.Grpc.Swagger
```

安装包后，我们可以配置启动项。首先需要将 Swagger 注册到 DI 容器中。

```csharp
builder.Services.AddGrpcSwagger();

// 注册 Swagger 生成器，定义一个或多个由 Swagger 生成器生成的文档
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1",
        new OpenApiInfo { Title = "gRPC transcoding", Version = "v1" });
});
```

然后我们需要启用中间件，将生成的 Swagger 文档作为 JSON 端点提供，并同时提供 Swagger UI。

```csharp
// 启用中间件，将生成的 Swagger 作为 JSON 端点提供
app.UseSwagger();

// 启用中间件，提供 swagger-ui（HTML、JS、CSS 等）
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
});
```

构建、运行并浏览到 /swagger，我可以测试我的 gRPC JSON 转码 HTTP API。

![Swagger UI](https://poornimanayar.co.uk/media/wkfe4dym/image.png)

好了，我们现在有了一个已经扩展为 HTTP API 的 gRPC API。当你想要将现有的 gRPC 服务扩展为同时具备 HTTP API 功能时，这是一个非常有用的特性。这个功能过去被称为 gRPC HTTP API，是一个实验性功能，但由于社区的极大兴趣，它现在已经成为 .NET 中的一个完整功能。

今天就到这里。期待与你再次相见。
