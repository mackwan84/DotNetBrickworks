# [.NET 中使用 Fusion Cache 进行缓存](https://www.nikolatech.net/blogs/fusion-cache-in-aspnetcore)

缓存是提升性能和减少应用程序负载的关键功能。

在复杂系统中，应用程序可能需要内存缓存和分布式缓存相互补充。

内存缓存可提高本地性能，而分布式缓存则支持扩展并确保系统中应用程序间的可用性。

在 .NET 中，我们有 **IDistributedCache** 和 **IMemoryCache**，但这些接口相对基础，缺乏几种高级功能。此外，并行管理它们可能会很有挑战性。

**HybridCache** 可能会在你的应用程序中替代这些接口，并解决一些缺失的功能。

然而，根据目前的见解，一旦它发布，很可能会是 FusionCache 的一个功能较弱的版本。

## FusionCache

**FusionCache** 是一个强大的缓存库，旨在处理数据缓存时提供高性能、弹性和灵活性。

它既用户友好，又提供了一系列高级功能：

- **L1+L2 缓存** - 你可以同时管理内存缓存和分布式缓存。
- **缓存雪崩** - 防止对相同数据的冗余并行请求，提高负载情况下的效率。
- **故障安全** - 一种允许你重用过期条目作为临时回退机制，以避免瞬时故障。
- **标记** - 支持对缓存项进行分组以便于管理，例如通过标记进行批量失效。
- **OpenTelemetry** - 对 OpenTelemetry 的原生支持。

这只是我过去使用过的一小部分功能。

如需完整列表，你可以在 GitHub 上查看。

## 缓存级别

- 第一级，L1（内存），将数据存储在应用程序的内存中，为频繁请求的数据提供快速访问，且开销最小。
- 第二级，L2（分布式），支持更大、可扩展的存储解决方案，例如 Redis。这一层确保数据可以在多个实例和环境中可用。

## 入门指南

要开始使用 **Fusion Cache**，首先需要安装必要的 NuGet 包。你可以通过 NuGet 包管理器执行此操作，或者在包管理器控制台中运行以下命令：

```bash
dotnet add package ZiggyCreatures.FusionCache
```

要开始使用 FusionCache，你需要将其注册到你的依赖注入设置中。这可以通过使用 **AddFusionCache** 方法简单地完成：

```csharp
builder.Services.AddFusionCache();
```

在这个示例中，我还会使用 Redis 作为我的分布式缓存。

### Redis

要使用 Redis，你只需添加 StackExchangeRedis 包，并像这样与 FusionCache 一起配置它：

```csharp
builder.Services
    .AddFusionCache()
    .WithDistributedCache(_ =>
    {
        var connectionString = builder.Configuration.GetConnectionString("Redis");
        var options = new RedisCacheOptions { Configuration = connectionString };

        return new RedisCache(options);
    })
    .WithSerializer(new FusionCacheSystemTextJsonSerializer());
```

一旦注册了 Redis，FusionCache 将将其用作二级缓存，并使用为其配置的序列化程序。

## 基本用法

FusionCache 提供了一组方法来管理缓存条目并执行各种操作。

以下是其方法的概述：

### SetAsync

当你想要设置新条目或更新现有条目时，可以使用 SetAsync 方法。

```csharp
public async Task<Result> Handle(Command request, CancellationToken cancellationToken)
{
    var product = await dbContext.Products
        .FirstOrDefaultAsync(x => x.Id == request.Id, cancellationToken);

    if (product is null)
    {
        return Result.NotFound();
    }

    product.Update(request.Name, request.Description, request.Price);

    await dbContext.SaveChangesAsync(cancellationToken);

    await cache.SetAsync(
        $"products-{product.Id}",
        product,
        token: cancellationToken);

    return Result.Success();
}
```

当你想要存储一个对象而不先检索它时，这是一个有用的方法。

### GetOrSetAsync

当你想要检索数据并在未找到时将其添加到缓存中，可以使用 GetOrSetAsync 方法。

```csharp
public async Task<Result<ProductResponse>> Handle(Query request, CancellationToken cancellationToken)
{
    var key = $"products-{request.Id}";

    var product = await cache.GetOrSetAsync(
        key,
        async _ =>
        {
            return await dbContext.Products
                .FirstOrDefaultAsync(
                    p => p.Id == request.Id,
                    cancellationToken);
        },
        token: cancellationToken);

    return product is null
        ? Result.NotFound()
        : Result.Success(product.Adapt<ProductResponse>());
}
```

### RemoveAsync

当缓存条目的基础数据在过期前发生变化时，你可以通过调用 RemoveAsync 并传入该条目的键来显式移除该条目。

```csharp
public async Task<Result> Handle(Command request, CancellationToken cancellationToken)
{
    var product = await dbContext.Products
        .FirstOrDefaultAsync(x => x.Id == request.Id, cancellationToken);

    if (product is null)
    {
        return Result.NotFound();
    }

    dbContext.Products.Remove(product);

    await dbContext.SaveChangesAsync(cancellationToken);

    var key = $"products-{product.Id}";

    await cache.RemoveAsync(key, token: cancellationToken);

    return Result.Success();
}
```

默认情况下，除非另有指定，所有方法都会同时更新 L1 和 L2 缓存。你可以配置缓存以独立控制每个缓存级别的行为，从而实现对缓存策略更精细的控制。

```csharp
public async Task<Result<Guid>> Handle(Command request, CancellationToken cancellationToken)
{
    var product = new Product(
        Guid.NewGuid(),
        DateTime.UtcNow,
        request.Name,
        request.Description,
        request.Price);

    dbContext.Products.Add(product);

    await dbContext.SaveChangesAsync(cancellationToken);

    var key = $"products-{product.Id}";

    await cache.SetAsync(
        key,
        product,
        token: cancellationToken,
        options: new FusionCacheEntryOptions(TimeSpan.FromMinutes(5))
            .SetSkipDistributedCache(true, true));

    return Result.Success(product.Id);
}
```

### HybridCache 抽象

HybridCache 是缓存实现的抽象，而 FusionCache 是首个可用于生产环境的实现。

它不仅是第一个第三方实现，更是总体上的第一个实现，甚至领先于微软自己的官方发布。

以下是如何使用带有 FusionCache 实现的 HybridCache 的示例：

```csharp
builder.Services
    .AddFusionCache()
    .WithDistributedCache(_ =>
    {
        var connectionString = builder.Configuration.GetConnectionString("Redis");
        var options = new RedisCacheOptions { Configuration = connectionString };

        return new RedisCache(options);
    })
    .WithSerializer(new FusionCacheSystemTextJsonSerializer())
    .AsHybridCache();
```

```csharp
internal sealed class Handler(
    IApplicationDbContext dbContext,
    HybridCache cache) : IRequestHandler<Query, Result<ProductResponse>>
{
    public async Task<Result<ProductResponse>> Handle(Query request, CancellationToken cancellationToken)
    {
        var key = $"products-{product.Id}";

        var product = await cache.GetOrCreateAsync(
            key,
            async _ =>
            {
                return await dbContext.Products
                    .FirstOrDefaultAsync(
                        p => p.Id == request.Id,
                        cancellationToken);
            },
            cancellationToken: cancellationToken);

        return product is null
            ? Result.NotFound()
            : Result.Success(product.Adapt<ProductResponse>());
    }
}
```

## 标记

从 v2 版本开始，Fusion Cache 还支持标记功能。标记可用于将缓存条目分组并一起使它们失效。

例如，你可以在调用 GetOrSetAsync 方法时设置标签：

```csharp
public async Task<Result<ProductResponse>> Handle(Query request, CancellationToken cancellationToken)
{
    string[] tags = ["products"];

    var key = $"products-{product.Id}";

    var product = await cache.GetOrSetAsync(
        key,
        async _ =>
        {
            return await dbContext.Products
                .FirstOrDefaultAsync(
                    p => p.Id == request.Id,
                    cancellationToken);
        },
        tags: tags,
        token: cancellationToken);

    return product is null
        ? Result.NotFound()
        : Result.Success(product.Adapt<ProductResponse>());
}
```

稍后你可以使用 RemoveByTagAsync 方法通过一个或多个标签来移除缓存条目：

```csharp
public async Task<Result> Handle(Command request, CancellationToken cancellationToken)
{
    await dbContext.Products.ExecuteDeleteAsync(cancellationToken);

    await cache.RemoveByTagAsync(CacheTags, token: cancellationToken);

    return Result.Success();
}
```

## 结论

Fusion Cache 的故事太过庞大，无法在一篇博文中完全讲述，因此我将在未来的文章中深入探讨其高级功能，并与 Microsoft 的 HybridCache 实现进行比较。

我第一次了解到 FusionCache 是在我需要为一个应用程序设计故障安全机制时，从那时起，它就成为了我首选的缓存库。

FusionCache 不仅是首个提供 L1+L2 缓存的解决方案，还包含了各种实用功能和高级弹性能力。

此外，我必须称赞 FusionCache 提供的出色文档，对于任何想要深入探索该库功能的人来说，这些文档都是完美的。

欢迎在 GitHub 上查看该项目并给它点个星：[Fusion Cache](https://github.com/ZiggyCreatures/FusionCache)

今天就到这里。期待与你再次相见。
