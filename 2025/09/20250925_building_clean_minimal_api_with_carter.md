# [使用 Carter 构建简洁且精简的 .NET API](https://stefandjokic.tech/posts/building-clean-minimal-api-with-carter)

## 背景

在现代 .NET 开发中，一直有向最小化 API 发展的趋势——摒弃控制器的臃肿，专注于真正重要的事情。但随着应用程序的增长，即使是最小化 API 也可能变得混乱且难以组织。

这就是 Carter 的用武之地。

Carter 为你提供了一种干净、模块化和可测试的方式来定义你的 API 端点——同时在底层仍然使用最小化 API。它轻量级、可组合，并且与你现有的.NET 技术栈完美配合。

今天，我们将逐步介绍：

- 为什么 Carter 很有用
- 如何在 .NET 项目中设置它
- 带有验证和映射的实际示例
- 使用 Carter 的优缺点
- 思考总结

## 为什么选择 Carter？

最小 API 非常适合小型项目 - 但随着项目的增长：

- 路由分散在多个文件中。
- 依赖注入逻辑被重复。
- 验证和映射逻辑使路由定义变得杂乱。
- 测试单个端点变得棘手。

Carter 帮助你将端点模块化为干净、可测试的组件 - 而无需回退到完整的 MVC 风格控制器。

## 在 .NET 中设置 Carter

首先安装 NuGet 包：

```bash
dotnet add package Carter
```

### 创建一个真实的 Carter API

让我们创建一个简单的产品 API，你可以：

- 创建一个产品
- 获取所有产品
- 使用 FluentValidation 进行验证
- 使用自定义映射器进行 DTO -> 实体映射

### 项目结构

```
MyApi/
├── Program.cs
├── Endpoints/
│   └── ProductModule.cs
├── Models/
│   ├── Product.cs
│   └── ProductDto.cs
├── Validators/
│   └── CreateProductDtoValidator.cs
```

Product.cs

```csharp
namespace MyApi.Models;

public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = default!;
    public decimal Price { get; set; }
}
```

ProductDto.cs

```csharp
namespace MyApi.Models;

public record CreateProductDto(string Name, decimal Price);
```

CreateProductDtoValidator.cs

```csharp
using FluentValidation;
using MyApi.Models;

namespace MyApi.Validators;

public class CreateProductDtoValidator : AbstractValidator<CreateProductDto>
{
    public CreateProductDtoValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Price).GreaterThan(0);
    }
}
```

ProductModule.cs

在 Carter 中，你无需直接在 Program.cs 中定义路由或使用控制器，而是创建将相关端点分组的"模块"。这些模块实现了 ICarterModule 接口。

这使得你的 API 具有模块化、简洁且易于维护的特点。

每个 Carter 模块都实现 ICarterModule 接口，该接口只需要一个方法：AddRoutes。

你可以直接将服务（如验证器）注入到处理程序中。

你将与"产品"相关的所有内容都保存在一个文件/模块中。

在底层，它最终都会编译为常规的最小 API 端点——只是组织得更好！

```csharp
using Carter;
using FluentValidation;
using Mapster;
using Microsoft.AspNetCore.Http.HttpResults;
using MyApi.Models;

namespace MyApi.Endpoints;

public class ProductModule : ICarterModule
{
    private static readonly List<Product> _products = [];

    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet("/products", () => _products);

        app.MapPost("/products", async (
            CreateProductDto dto,
            IValidator<CreateProductDto> validator) =>
        {
            var validationResult = await validator.ValidateAsync(dto);
            if (!validationResult.IsValid)
            {
                var errors = validationResult.Errors
                    .ToDictionary(e => e.PropertyName, e => e.ErrorMessage);
                return Results.BadRequest(errors);
            }

            var product = dto.ToProduct<Product>();
            product.Id = Guid.NewGuid();
            _products.Add(product);

            return Results.Created($"/products/{product.Id}", product);
        });
    }
}
```

### Program.cs

就是这样！你现在拥有一个包含验证和映射的简洁 API - 所有路由都已组织到模块中。

```csharp
using Carter;
using FluentValidation;
using MyApi.Models;
using MyApi.Validators;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCarter();
builder.Services.AddScoped<IValidator<CreateProductDto>, CreateProductDtoValidator>();

var app = builder.Build();

app.MapCarter(); // 自动注册映射所有 Carter 模块

app.Run();
```

## 为什么开发者喜欢 Carter

### 保持内容整洁有序

与其将数十个路由堆砌在你的 Program.cs 中，Carter 让你能够将相关的端点整理到简洁的模块中。这种感觉就是更好。

### 与你的项目共同成长

起初一个小型 API 可能很快就会变得混乱。即使项目不断增长，Carter 也能为你的项目提供结构。

### 你可以注入任何你需要的东西

需要一个服务？一个验证器？一个记录器？只需直接注入到你的路由中 - 无需额外设置，无需控制器样板代码。

### 与现代模式完美契合

如果你喜欢垂直切片架构、CQRS 或最小 API - Carter 与这些风格搭配得非常好。

### 更简洁的启动

你的 Program.cs 保持简短整洁。只需调用 app.MapCarter()，所有模块就会自动连接起来。

### 快速且轻量

没有控制器或属性开销。它只是极简 API - 但组织得更好。

### 易于测试

由于模块是简单的类，它们非常易于编写测试 - 背后没有任何魔法或反射。

### 灵活且即插即用

使用你喜欢的任何工具 - FluentValidation、Mapster、MediatR、Dapper、EF Core… Carter 不会妨碍你。

## 注意事项

### 社区较小，文档较少

Carter 很棒，但它不如.NET MVC 主流。你可能无法在 Stack Overflow 上那么快找到答案。

### 你手动定义路由

这里没有 \[HttpGet\]、\[Route\] 或基于属性的路由 - 只有像 MapGet 这样的常规方法调用。有些人怀念那些属性。

### 没有花哨的模型绑定属性

你不会看到 \[FromBody\] 或 \[FromQuery\]。绑定方式就像在最小 API 中一样 - 简洁，但如果你习惯了 MVC，则会有些不同。

### 你必须决定如何组织结构

Carter 为你提供了灵活性，但随之而来的是责任。你需要为模块、验证器等创建自己的结构。

### 没有内置的过滤器或属性

如果你习惯使用 \[Authorize\]、\[ValidateModel\] 或操作过滤器，你将需要自己实现这些行为。

## 总结

Carter 是那种会让你思考"为什么我没有早点使用它？"的库之一。它兼具最小 API 的灵活性，并添加了恰到好处的结构 - 不会让你回到臃肿的控制器和特性的世界。

如果你正在构建的 API 开始超出几个路由，或者你只是想要一种更清晰的方式来组织你的功能，那么 Carter 绝对值得一试。

它简单、轻量级，并且能完美融入现代 .NET 项目。

当然，它不像 MVC 那样主流，你需要自己定义一些结构 - 但正是这种自由使其如此强大。

所以下次当你发现自己深陷于混乱的 Program.cs 文件，不知道该把那个新端点放在哪里时... 也许可以尝试一下 Carter。

你可能会爱上你的代码开始变得如此整洁的感觉。

今天就到这里。期待与你再次相见。
