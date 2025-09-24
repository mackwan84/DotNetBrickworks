# [.NET 世界中的 gRPC 服务器反射](https://martinbjorkstrom.com/posts/2020-07-08-grpc-reflection-in-net)

> gRPC 服务器反射是服务器的可选扩展，用于帮助客户端在运行时构建请求，而无需将存根信息预编译到客户端中。
> —— gRPC 服务器反射协议

gRPC 服务器反射的主要用例是调试和测试工具。可以将其视为 gRPC 版本的 cURL 或 Postman。但还有另一个有趣的用例，特别是在 .NET 世界中，我们通过 protobuf-net.Grpc 拥有 Code First 支持。

如果你使用过 WCF，你可能熟悉一种代码优先的方法，该方法通过 MEX 或 WSDL 端点公开关于服务的元数据。然后可以使用这些元数据端点来生成客户端存根。此外，如果你创建了一个通过 HTTP 使用的标准 SOAP 服务，通过使用 WSDL 文档，也可以实现与其他编程语言的互操作性。这与我们今天许多人使用 ASP.NET Core 和 OpenAPI 所做的类似。我们在 C#中创建控制器和模型，使用 Swashbuckle 生成 OpenAPI 定义，然后使用 Autorest 在 TypeScript 中生成客户端。

虽然 Google 处理 protobuf 和 gRPC 的方式一直是契约优先，但 Marc Gravell 的 protobuf-net 始终将 Code First 开发作为首要任务。使用 Google 的 protobuf，你以 .proto 格式定义服务和消息，然后使用 protobuf 编译器（protoc）在你选择的语言中生成消息、客户端和服务器存根。然而，使用 protobuf-net，你可以直接从 C#、F# 或 VB.NET 代码开始。不需要 .proto 文件。如果你不需要与 .NET 生态系统之外的其他语言进行互操作，protobuf-net 的代码优先方式非常棒，即使你需要，也有方法可以为消息类型生成 .proto 格式的契约。

protobuf-net.Grpc 现在支持为服务类型生成 proto 契约，并支持 gRPC 服务器反射。

## 使用 gRPC 服务器反射从 Code First 服务生成 .PROTO 文件

让我们来看看如何使用 protobuf-net.gRPC 来实现 gRPC 服务器反射，以及如何使用命令行客户端生成 .proto 文件。我们首先创建一个新的 ASP5 Core 应用程序并安装所需的 NuGet 包：

```bash
dotnet new web -n Reflection.Sample

cd Reflection.Sample

dotnet add package protobuf-net.Grpc.AspNetCore
dotnet add package protobuf-net.Grpc.AspNetCore.Reflection
```

接下来，我们创建一个简单的服务和一些消息。

```csharp
using System.ServiceModel;
using System.Threading.Tasks;
using ProtoBuf;

namespace Reflection.Sample
{
    [ServiceContract]
    public interface IGreeter
    {
        ValueTask<HelloReply> SayHello(HelloRequest request);
    }

    [ProtoContract]
    public class HelloRequest
    {
        [ProtoMember(1)]
        public string Name { get; set; }
    }

    [ProtoContract]
    public class HelloReply
    {
        [ProtoMember(1)]
        public string Message { get; set; }
    }

    public class Greeter : IGreeter
    {
        public ValueTask<HelloReply> SayHello(HelloRequest request)
        {
            return new ValueTask<HelloReply>(new HelloReply
            {
                Message = "Hello " + request.Name
            });
        }
    }
}
```

然后在 Startup.cs 中注册我们的服务

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ProtoBuf.Grpc.Server;

namespace Reflection.Sample
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddCodeFirstGrpc();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGrpcService<Greeter>();
            });
        }
    }
}
```

我们需要做的最后一件事是启用 gRPC 服务器反射。这也是在我们的 Startup.cs 中完成的

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ProtoBuf.Grpc.Server;

namespace Reflection.Sample
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddCodeFirstGrpc();
            services.AddCodeFirstGrpcReflection();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGrpcService<Greeter>();
                endpoints.MapCodeFirstGrpcReflectionService();
            });
        }
    }
}
```

## 使用反射端点

gRPC 提供了自己的 CLI 来使用服务器反射 API（<https://github.com/grpc/grpc/blob/master/doc/command_line_tool.md>），但它存在一些问题。你需要从源代码构建它（至少我找不到 Windows 二进制文件），而且它不支持生成 .proto 文件。考虑到这一点，我决定为此创建一个 .NET 全局工具。

我们首先从 NuGet 安装全局工具。

```bash
dotnet tool install -g dotnet-grpc-cli
```

随着我们新创建的 gRPC 服务运行，我们将运行该工具来检查有哪些服务可用：

```bash
dotnet grpc-cli ls https://localhost:5001

Reflection.Sample.Greeter
grpc.reflection.v1alpha.ServerReflection
```

我们可以看到我们创建的 Greeter 服务，以及服务器反射服务。正如你所见，反射服务只是另一个 gRPC 服务。如果你好奇，可以在这里找到 gRPC 反射服务的 proto 定义。

既然我们知道了服务的名称，就可以列出这些方法。

```bash
dotnet grpc-cli ls https://localhost:5001 Reflection.Sample.Greeter

filename: Reflection.Sample.Greeter.proto
package: Reflection.Sample
service Greeter {
  rpc SayHello(Reflection.Sample.HelloRequest) returns (Reflection.Sample.HelloReply) {}
}
```

同时以.proto 格式输出结果

```bash
dotnet grpc-cli dump https://localhost:5001 Reflection.Sample.Greeter

---
File: Reflection.Sample.HelloRequest.proto
---
syntax = "proto3";
package Reflection.Sample;

message HelloRequest {
  string Name = 1;
}

---
File: Reflection.Sample.HelloReply.proto
---
syntax = "proto3";
package Reflection.Sample;

message HelloReply {
  string Message = 1;
}

---
File: Reflection.Sample.Greeter.proto
---
syntax = "proto3";
import "Reflection.Sample.HelloRequest.proto";
import "Reflection.Sample.HelloReply.proto";
package Reflection.Sample;

service Greeter {
   rpc SayHello(HelloRequest) returns (HelloReply);
}
```

我们可能希望将其写入文件系统，所以我们将运行以下命令。

```bash
dotnet grpc-cli dump https://localhost:5001 Reflection.Sample.Greeter -o ./protos
```

就这样，我们现在有了从服务器反射端点生成的 .proto 文件。这些 .proto 文件现在可用于以你选择的语言生成客户端存根。

## 总结

如你所见，gRPC 服务器反射在代码优先场景中非常有用，但它对于测试和调试也很有用。gRPC CLI 支持调用方法，我也计划在 .NET 全局工具中支持这一功能。还值得一提的是，Protogen（一个从 proto 文件生成 protobuf-net 源代码的工具）最近也通过新的 --grpc 选项支持从 gRPC 服务器反射端点生成源代码。

在我看来，使用 protobuf-net.Grpc 的代码优先方式是将旧的 WCF 服务迁移到 .NET Core 和 gRPC 的最简单、最直接的方法。它比官方文档上使用 Google protobuf 而非 protobuf-net 的官方指南要简单得多。使用 protobuf-net.Grpc，你很可能可以保留所有数据契约不变，而只需要在数据成员属性中添加 Order 属性。然而，你的服务契约将需要一些重构。但最终，与将数据契约和服务契约迁移到 .proto 格式相比，这些是最小的改动。如果你还需要与其他编程语言的互操作性，从你的 gRPC 服务生成 .proto 文件是小菜一碟。那么，你还在等什么？现在就开始将你的 WCF 服务迁移到 .NET Core，并确保在迁移时使用 protobuf-net.Grpc。

今天就到这里。期待与你再次相见。
