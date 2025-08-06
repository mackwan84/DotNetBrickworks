# [.NET 中如何实现缓存策略](https://antondevtips.com/blog/how-to-implement-caching-strategies-in-dotnet)

缓存是一种提高应用程序性能和可扩展性的强大技术。在 .NET 中，我们通常使用内存缓存或像 Redis 这样的分布式缓存。

根据读写频率、数据新鲜度和一致性要求，你可以考虑使用不同的**缓存策略**。

今天，我想向你展示 5 种缓存策略及其在 .NET 中的实现。我们将讨论每种策略的用例，以及每种选项的优缺点。

在所有代码示例中，我将使用 .NET 9 中提供的新 HybridCache ，但如果你更喜欢直接使用 IMemoryCache 、 IDistributedCache 或任何 Redis SDK，也可以使用相同的代码。

## 在 ASP.NET Core 应用程序中设置 HybridCache

在使用缓存时，我们必须意识到缓存雪崩问题：当多个请求可能会遇到缓存未命中，并且都会调用数据库来检索实体。而不是只调用一次数据库来获取实体并将其写入缓存。

你可以引入手动锁定来解决这个问题，但这种方法扩展性不太好。

这就是为什么我正在使用 .NET 9 中提供的一个新的缓存库 HybridCache ，它可以防止缓存雪崩问题。

HybridCache 是 IMemoryCache 和 IDistributedCache 的优秀替代品，因为它将两者结合在一起，并且可以与以下内容一起使用：

- 内存缓存
- 像 Redis 这样的分布式缓存

以下是 HybridCache 的工作原理：

- 首先检查项目是否存在于内存缓存中（一级缓存）
- 如果未找到项目 - 则回退到在分布式缓存（如 Redis）中搜索（二级缓存）
- 如果未找到项目 - 则回退到回调方法并从数据库获取值（你需要提供一个委托）

要开始使用 HybridCache ，请添加以下 Nuget 包：

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

以下是在 DI 中注册它的方法：

```csharp
#pragma warning disable EXTEXP0018
services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromMinutes(10)
    };
});
#pragma warning restore EXTEXP0018
```

你需要使用 pragma 禁用该警告，因为 HybridCache 目前处于预览状态（当你阅读本文时可能已经发布）。

默认情况下，它启用了内存缓存。

以下是在 DI 中注册 Redis 的方法，它将被 HybridCache 自动识别：

```csharp
services
    .AddStackExchangeRedisCache(options =>
    {
        options.Configuration = configuration.GetConnectionString("Redis")!;
    });
```

现在让我们深入探讨缓存策略。

## 旁路缓存 Cache Aside

### 工作原理

- 应用程序在读取数据时首先检查缓存。
- 在缓存未命中时，数据从数据库中获取并添加到缓存中。
- 后续读取使用新缓存的数据。

### 最适合

- 当数据经常被读取但可以偶尔处理从数据库获取数据的情况时。

### 类比

- 首先检查你的图书馆（缓存）中是否有这本书，如果找不到这本书，你就去书店（数据库）寻找，然后重新填满你的图书馆。

这是在代码中的实现方式：

```csharp
public class CacheAsideProductService(IMemoryCache cache, IProductRepository repository)
    : IProductService
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        // 定义缓存键
        var cacheKey = $"product:{id}";

        // 1. 尝试从缓存中获取产品
        var product = cache.Get<Product>(cacheKey);
        if (product is not null)
        {
            return product; // 如果找到，直接返回缓存中的数据
        }

        // 2. 如果未找到，从数据库加载产品
        product = await repository.GetByIdAsync(id);
        if (product != null)
        {
            // 3. 更新缓存
            cache.Set(cacheKey, product, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });
        }

        return product;
    }
}
```

在此示例中，我们仅在缓存为空时才查询数据库。这种方法使缓存保持"按需"更新，是最广泛使用的模式之一。

### 优点

- 实现简单。
- 减少后续读取时的数据库负载。
- 数据仅在需要时被缓存（按需）。

### 缺点

- 数据更改后首次请求时可能会出现缓存未命中（如果缓存未预先填充）。
- 需要在应用程序中添加额外的逻辑来处理在缓存中获取和存储数据。

### 实际应用案例

电商产品目录：

- 场景：在在线商店中，产品详情（名称、价格、描述）的读取频率远高于其变更频率。
- 为何选择旁路缓存？你可以在请求产品详细信息时将其延迟加载到缓存中，从而减少填充缓存的初始开销。如果数据发生变化，移除或使缓存条目失效相对简单，下一次读取会导致重新加载。
- 好处：只有实际被查看的产品才会被缓存，既节省了内存，又加快了对热门商品的重复查询速度。

博客网站：

- 场景：文章和博客帖子不经常更改，但被大量阅读。
- 为何采用旁路缓存？当用户首次请求一篇新文章时，会从源获取该文章并进行缓存，从而使后续的检索几乎瞬间完成。

## 直读缓存 Read Through Cache

### 工作原理

- 缓存本身负责在缓存未命中时从数据存储中获取数据。
- 你的应用程序只需与缓存层交互，当缓存未命中时，它会"直读"数据库。

### 最适合

- 频繁访问的数据。
- 降低应用程序中"数据来源"的复杂性。

### 类比

- 想象一下办公室里的一台咖啡机，当咖啡豆（数据库）用完时，它会自动补充。作为用户，你只需与咖啡机（缓存）交互来获取咖啡，完全无需担心咖啡豆来自何处或如何重新补充。

这是在代码中的实现方式：

```csharp
public class ReadThroughCacheProductService(HybridCache cache, IProductRepository repository)
    : IProductService
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        // 定义缓存键
        var cacheKey = $"product:{id}";

        // 实现直读缓存
        var product = await cache.GetOrCreateAsync<Product?>(
            cacheKey,
            // 数据加载函数，从数据库中获取
            async token => await repository.GetByIdAsync(id),
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(10)
            });

        return product;
    }
}
```

在这里，HybridCache 负责检查产品是否在缓存中。如果不在，它会自动调用 IProductRepository.GetByIdAsync 并缓存结果。这将缓存逻辑抽象到缓存层本身，简化了应用程序代码。

### 优点

- 将缓存逻辑集中在一个地方（缓存层）。
- 通过抽象化数据库获取操作来简化应用程序代码。
- 确保数据加载的一致性和无缝性。

### 缺点

- 缓存库或服务必须支持直读功能。
- 可能比旁路缓存模式更复杂的配置或设置。
- 应用程序对数据库读取的直接控制较少。

### 实际应用案例

产品推荐：

- 场景：个性化产品推荐依赖于机器学习模型或高级查询。
- 为什么要使用直读？与其编写"从机器学习服务获取然后放入缓存"的逻辑，你可以让缓存来管理这些获取操作。当推荐条目不在缓存中时，缓存层会自动调用模型或专门的服务。
- 优点：保持应用程序代码整洁，并将"未命中时加载"的逻辑集中在缓存层中。

全局配置/功能标志：

- 场景：应用程序可能每次需要检查某个功能是否启用时，都会读取配置标志（用于 A/B 测试或功能开关）。
- 为什么要使用直读模式？如果标志不在缓存中，系统会自动从配置存储中检索它，确保始终提供最新设置，同时最小化应用程序代码开销。

## 绕写缓存 Write Around Cache

### 工作原理

- 数据直接写入数据库，绕过缓存。
- 缓存仅在下次读取操作发生时才会更新（后续读取时发生缓存未命中）。

它与旁路缓存模式有何不同？这两种策略在读取方面相似（首先访问缓存，未命中时回退到数据库），但区别在于：

- **Cache Aside**：写入操作也可以立即更新或使缓存失效，确保一致性。
- **Write Around**：写入操作直接到数据库，缓存仅在下次读取时更新。

### 最适合

- 写入密集型系统，在每次写入时立即更新缓存可能会非常昂贵。
- 数据频繁变化但读取请求较少，或者可以容忍数据在下一次读取之前保持陈旧的情况。

### 类比

- 当你为图书馆（数据库）购买一本新书时，你不会立即更新图书馆的公共目录（缓存）。相反，只有当下一次有人询问那本书时，目录才会被更新。

这是在代码中的实现方式：

```csharp
public class WriteAroundCacheProductService(IMemoryCache cache, IProductRepository repository)
    : IProductService
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        // 定义缓存键
        var cacheKey = $"product:{id}";

        // 1. 尝试从缓存中获取产品
        var product = cache.Get<Product>(cacheKey);
        if (product is not null)
        {
            return product; // 缓存命中
        }

        // 2. 如果未找到，从数据库加载产品
        product = await repository.GetByIdAsync(id);
        if (product != null)
        {
            // 3. 更新缓存
            cache.Set(cacheKey, product, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            });
        }

        return product;
    }

    public async Task AddAsync(Product product)
    {
        // 将产品添加到数据库。缓存未更新
        await repository.AddAsync(product);
    }
}
```

使用 Write Around 策略时，你在写入操作时会跳过缓存，这可以减少缓存失效的开销。代价是可能会提供过时数据，直到下一次读取操作强制从数据库获取数据。

读取值时，你可以使用 Cache Aside 或 Read Through Cache 策略。Write Around 策略通常与这两种策略之一结合使用。

### 优点

- 减少频繁写入的开销（无需在每次写入时更新缓存）。
- 如果已经在读取操作中使用了缓存旁路方法，则实现起来很简单。
- 有助于避免在写入时频繁使缓存失效。

### 缺点

- 从缓存中提供可能过时的数据，直到下一次读取触发更新。
- 如果数据经常更新但也被频繁读取，缓存可能会经常变得不同步。
- 如果与其他模式混合使用，读取逻辑会稍微复杂一些。

### 实际应用案例

日志与分析管道：

- 场景：大量日志持续写入，但通常以批处理方式进行分析，或仅在特定查询时才进行分析。
- 为何采用绕写策略？直接将日志写入数据存储（如 NoSQL 数据库或日志服务器）更快更简单，并且仅在读取数据进行分析时才进行缓存。这样可以避免为每个日志条目更新缓存所带来的开销。

高频数据摄入：

- 场景：物联网设备生成大量传感器读数并存储在时间序列数据库中，但读取操作是偶发性的（例如，用于历史分析）。
- 为何使用写绕写策略？为每次传感器读数不断更新缓存是浪费的，特别是当这些读数很少被查询时。写绕过确保数据库始终拥有最新数据，而缓存仅在真正的读取请求时才会填充。

## 写回缓存 Write Back Cache

### 工作原理

- 数据首先被写入缓存。
- 缓存在特定条件或时间间隔后，将数据异步写回数据库。

### 最适合

- 高速、写入密集型场景。
- 可以处理最终一致性的应用程序（数据库可能会暂时落后于缓存）。

### 类比

- 快速将电话号码记在一张纸上（缓存），然后在有更多时间时将其添加到手机的联系人列表中（数据库）。

这是在代码中的实现方式：

```csharp
public class WriteBackCacheProductCartService(
    HybridCache cache,
    IProductCartRepository repository,
    IProductRepository productRepository,
    Channel<ProductCartDispatchEvent> channel)
{
    public async Task<ProductCartResponse?> GetByIdAsync(Guid id)
    {
        // 定义缓存键
        var cacheKey = $"productCart:{id}";

        // 获取 ProductCart（如果缓存中不存在则从数据库中获取）
        var productCartResponse = await cache.GetOrCreateAsync<ProductCartResponse?>(
            cacheKey,
            // 数据加载器函数，从数据库中获取
            async token => 
            {
                var productCart = await repository.GetByIdAsync(id);
                return productCart?.MapToProductCartResponse();
            },
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(10)
            });

        return productCartResponse;
    }

    public async Task<ProductCartResponse> AddAsync(ProductCartRequest request)
    {
        var productCart = new ProductCart
        {
            Id = Guid.NewGuid(),
            UserId = request.UserId,
            CartItems = request.ProductCartItems.Select(x => x.MapToCartItem()).ToList()
        };
        
        var cacheKey = $"productCart:{productCart.Id}";

        // 添加或更新缓存中的 ProductCart
        var productCartResponse = productCart.MapToProductCartResponse();
        await cache.SetAsync(cacheKey, productCartResponse);

        // 将 ProductCart 排队以进行数据库同步
        await channel.Writer.WriteAsync(new ProductCartDispatchEvent(productCart, false));

        return productCartResponse;
    }
}
```

当创建一个新的 ProductCart 时，它会立即添加到缓存中，然后在另一个线程中异步写入数据库。

这里我使用了一个有界的 Channel ，它会触发一个关于已创建 ProductCart 的异步进程内事件：

```csharp
var channelOptions = new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
};

services.AddSingleton(_ => Channel.CreateBounded<ProductCartDispatchEvent>(channelOptions));
```

我有一个 BackgroundService，它从 Channel 读取事件并异步更新数据库：

```csharp
public record ProductCartDispatchEvent(ProductCart ProductCart, bool IsDeleted);

public class WriteBackCacheBackgroundService(IServiceScopeFactory scopeFactory,
    Channel<ProductCartDispatchEvent> channel) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var command in channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = scopeFactory.CreateScope();
            var repository = scope.ServiceProvider.GetRequiredService<IProductCartRepository>();
            
            if (command.IsDeleted)
            {
                await repository.DeleteAsync(command.ProductCart.Id);
                return;
            }

            var existingCart = await repository.GetByIdAsync(command.ProductCart.Id);
            if (existingCart is null)
            {
                await repository.AddAsync(command.ProductCart);
            }
            else
            {
                existingCart.CartItems = command.ProductCart.CartItems;
                await repository.UpdateAsync(existingCart);
            }
        }
    }
}
```

这种模式可以显著提高写入速度，但需要强大的冲突解决和故障处理机制来保证一致性。

### 优点

- 非常快速的写入（数据库不会立即被访问）。
- 显著减少数据库的写入负载。
- 可以在高写入场景中提高应用程序吞吐量。

### 缺点

- 如果缓存节点在将更改刷新到数据库之前发生故障，则存在数据丢失的风险。
- 更复杂的同步逻辑（你需要一个后台进程或事件来处理数据库更新）。
- 在数据写回之前，缓存与数据库之间存在不一致性。

### 实际应用案例

电子商务中的购物车：

- 场景：用户频繁地在其在线购物车中添加或移除商品。最终状态只有在结账时才真正关键。
- 为什么使用写回？写入操作可以非常快，因为它们首先命中缓存，系统可以定期将更新刷新到数据库。这使得在购物高峰期间能够实现高写入吞吐量，而数据库最终会获取到最终的购物车详细信息。
- 警告：确保在缓存失败时购物车状态永不丢失，需要强大的复制或备份机制。

在线游戏会话状态：

- 场景：多人游戏通常需要处理频繁的玩家状态变化（分数、物品等）。实时性能至关重要。
- 为何使用写回策略？你可以即时更新内存或分布式缓存以实现快速游戏体验。后台作业会按计划或在会话结束时将更改推送到持久存储（数据库）。
- 警告：如果缓存意外宕机，除非通过复制得到良好保护，否则最近的更改可能会丢失。

## 直写缓存 Write Through Cache

### 工作原理

- 数据会在单次操作中同时写入缓存和数据库。
- 确保缓存和数据库立即保持同步。

### 最适合

- 对于一致性至关重要的系统，读取操作必须立即看到最新的更新
- 可以接受写入两处开销的环境。

### 类比

- 在纸质计划本（数据库）中记录日记条目的同时，更新共享的数字日历（缓存）以保持两者同步。

这是在代码中的实现方式：

```csharp
public class WriteThroughCacheProductService(HybridCache cache, IProductRepository repository) : IProductService
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        // 定义缓存键
        var cacheKey = $"product:{id}";

        // 缓存中应该存在这个值，但以防万一，我们可以在缓存未命中时检查数据库
        var product = await cache.GetOrCreateAsync<Product?>(
            cacheKey,
            // 数据加载器函数，从数据库中获取
            async token => await repository.GetByIdAsync(id),
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(10)
            });

        return product;
    }

    public async Task AddAsync(Product product)
    {
        // 将产品添加到数据库
        await repository.AddAsync(product);

        // 将产品写入缓存（写透）
        var cacheKey = $"product:{product.Id}";
        
        await cache.SetAsync(cacheKey, product, new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(10)
        });
    }
}
```

在这里，每次你保存（更新）产品时，都会同时在缓存和数据库中进行操作，确保立即一致性。

直写策略通常也与旁路缓存或直读缓存策略在读取端结合使用。

### 优点

- 强一致性 — 缓存和数据库始终保持同步。
- 读取逻辑更简单（始终信任缓存）。
- 理想用于不能承担不一致风险的关键数据。

### 缺点

- 每次写入的成本更高，因为它同时访问缓存和数据库。
- 如果缓存或数据库中的任何一个速度较慢，写入操作可能会成为瓶颈。
- 不会减少数据库的写入负载。

### 实际应用案例

金融交易：

- 场景：银行、交易或支付系统，这些系统不能承担数据过时或丢失的风险（例如账户余额）。
- 为何采用直写策略？每次写入操作会同时更新缓存和数据库，确保余额或交易记录等关键数据的绝对一致性。这种性能开销的代价是由准确性要求所决定的。

具有实时准确性的库存：

- 场景：某些电商或零售系统需要实时准确的库存计数（例如，用于即时库存管理）。
- 为何使用直写策略？当商品售出时，你会同时更新数据库和缓存。这样，正确的库存计数会立即可用于各处，从而防止超卖或并发问题。

## 总结

让我们回顾一下缓存策略：

- **旁路缓存（Cache Aside）**：缓存未命中时延迟加载缓存条目，广泛用于灵活的读取场景。
- **直读缓存（Read Through）**：缓存自动处理从数据库加载数据。
- **绕写缓存（Write Around）**：写入操作直接进入数据库，缓存仅在下次读取时更新。
- **写回缓存（Write Back）**：写入操作首先进入缓存，然后数据库稍后异步更新。
- **直写缓存（Write Through）**：写入操作同时更新缓存和数据库。

选择正确的策略：

- 如果你想要简单性和典型方法：Cache Aside。
- 如果你的缓存层能够自主加载数据，降低应用程序复杂性：请使用 Read Through。
- 如果你的系统是写密集型的且不需要即时一致性：使用 Write Around 或 Write Back 策略。
- 如果缓存和数据库的即时一致性至关重要：使用 Write Through 策略。

根据以下因素仔细选择缓存策略：

- 读写混合
- 性能要求
- 一致性要求

今天就到这里。期待与你再次相见。
