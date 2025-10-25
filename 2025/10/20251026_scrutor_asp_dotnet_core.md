# [使用 Scrutor 在 ASP.NET Core 中实现更好的依赖注入](https://www.nikolatech.net/blogs/scrutor-asp-dotnet-core)

依赖注入（DI）是 ASP.NET Core 应用程序的基石。

依赖注入是一种实现控制反转（IOC）原则的软件设计模式，它允许对象从外部接收其依赖项，而不是自己创建它们。

虽然内置的 DI 容器功能强大，但随着项目规模的增长，手动管理注册可能会变得重复且容易出错。

这就是 Scrutor 发挥作用的地方，它扩展了 Microsoft.Extensions.DependencyInjection 包。

## 什么是 Scrutor？

在 ASP.NET Core 中，你注册的每个服务都有一个生命周期，用于控制服务实例的存活时间。主要有三种生命周期：

- Transient：每次请求都会创建一个新的实例。
- Scoped：在每个请求范围内创建一个实例。
- Singleton：应用程序生命周期内只创建一个实例。

随着应用程序的增长，你创建的服务数量也会增加，每次都必须在 DI 容器中手动注册它们。错误和遗漏的注册会变得更加频繁，特别是当服务数量不断增加时。

更别提服务装饰会变得多么繁琐了。

这就是 Scrutor 发挥作用的地方。

Scrutor 是一个用于 ASP.NET Core 的开源库，它扩展了默认的 DI 容器。它允许开发者：

- 扫描程序集并自动注册服务。
- 装饰服务而无需修改现有注册。
- 减少 DI 配置的样板代码。

简而言之，Scrutor 让你能用更少的代码实现更多的功能，同时保持你的依赖注入设置简洁、可维护和可扩展。

顺便说一下，我知道你们中的一些人可能在想："Scrutor 是否使用了反射？"

答案是肯定的。近年来，确实存在避免反射的趋势，但由于这只发生在应用程序启动时，所以并不是什么大问题。很可能没有人会注意到任何性能差异，因此无需担心。

## 入门指南

要开始使用 Scrutor，你首先需要安装必要的 NuGet 包。你可以通过 NuGet 包管理器执行此操作，或者在包管理器控制台中运行以下命令：

```bash
Install-Package Scrutor
```

添加该包后，你的 IServiceCollection 将获得两个强大的扩展方法：

- `Scan`：用于扫描程序集并根据约定注册服务。
- `Decorate`：用于装饰现有服务。

### Scan 方法

而不是像这样手动注册服务：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTransient<IMyService, MyService>();
builder.Services.AddTransient<IOtherService, OtherService>();
builder.Services.AddTransient<IAnotherService, AnotherService>();
// 其他服务注册...
```

你可以让 Scrutor 自动处理。使用 Scrutor，只需几行代码就能扫描程序集并注册所有服务，大大减少了样板代码和忘记注册的可能性：

```csharp
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses()
    .AsImplementedInterfaces()
    .WithTransientLifetime());
```

这将自动找到程序集中实现接口的所有类，并以瞬态生命周期注册它们。

在我的应用程序中，我使用标记接口方法来一致地分配适当的生命周期。

为每个生命周期创建标记接口：

```csharp
public interface ITransient { }
public interface IScoped { }
public interface ISingleton { }
```

将它们添加到你的接口中：

```csharp
public interface IMyService : IScoped
{
    // ...
}

public sealed class MyService : IMyService
{
    // ...
}
```

创建按生命周期注册服务的扩展方法：

```csharp
internal static class ServiceCollectionExtensions
{
    internal static IServiceCollection AddTransientServicesAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<ITransient>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsMatchingInterface()
                .WithTransientLifetime());

    internal static IServiceCollection AddScopedServicesAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<IScoped>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsMatchingInterface()
                .WithScopedLifetime());

    internal static IServiceCollection AddSingletonServicesAsMatchingInterfaces(
        this IServiceCollection services, Assembly assembly) =>
        services.Scan(scan =>
            scan.FromAssemblies(assembly)
                .AddClasses(filter => filter.AssignableTo<ISingleton>(), false)
                .UsingRegistrationStrategy(RegistrationStrategy.Throw)
                .AsMatchingInterface()
                .WithSingletonLifetime());
}
```

在启动时调用扩展方法：

```csharp
builder.Services
    .AddTransientServicesAsMatchingInterfaces(assembly)
    .AddScopedServicesAsMatchingInterfaces(assembly)
    .AddSingletonServicesAsMatchingInterfaces(assembly);
```

就这样，你的所有服务都会以正确的生命周期自动注册，不再有手动错误或遗漏的注册。

### Decorate 方法

装饰模式是一种结构型设计模式，它允许你在不修改对象原始实现的情况下，动态地为对象添加行为。

简而言之，这是一种用额外功能"包装"服务的方法，类似于服务的中间件。

以下是不使用 Scrutor 的服务装饰示例：

```csharp
services.AddScoped<IMyService, MyService>();
services.AddScoped<IMyService>(provider =>
{
    var original = provider.GetRequiredService<MyService>();
    return new LoggingMyService(original);
});
```

使用 Scrutor，你可以简化此过程：

```csharp
services.AddScoped<IMyService, MyService>();
services.Decorate<IMyService, LoggingMyService>();
```

现在想象一下手动使用多个装饰器，情况很快就会变得混乱：

```csharp
services.AddScoped<IMyService, MyService>();
services.AddScoped<IMyService>(provider =>
{
    var original = provider.GetRequiredService<MyService>();
    return new LoggingMyService(original);
});
services.AddScoped<IMyService>(provider =>
{
    var previous = provider.GetRequiredService<MyService>();
    return new CachingMyService(previous);
});
```

使用 Scrutor，你可以轻松地链式调用装饰器：

```csharp
services.AddScoped<IMyService, MyService>();
services.Decorate<IMyService, LoggingMyService>();
services.Decorate<IMyService, CachingMyService>();
```

一切保持可读性，添加新的装饰器只需一行，不需要 lambda 样板代码。

注意：顺序很重要，第一个装饰器包装原始服务，下一个装饰器包装前一个装饰器，以此类推。

## 总结

Scrutor 扩展了默认的 DI 容器，允许开发者扫描程序集、自动注册服务并清晰地装饰它们。

通过减少样板代码并保持生命周期一致，Scrutor 有助于维护可扩展、可维护且可读的 DI 设置。

今天就到这里。希望这些内容对你有帮助。

如果你有任何问题或想分享你使用 Scrutor 的经验，请在评论区留言！
