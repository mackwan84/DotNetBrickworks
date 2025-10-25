# [内存缓存与分布式缓存——哪种更能为你的外卖应用提供动力？](https://medium.com/@dotnetfullstackdev/in-memory-cache-vs-distributed-cache-which-one-powers-your-system-better-40daa3236271)

在现代应用中，每一毫秒都至关重要。这就是为什么缓存是最强大的性能提升器之一：它存储频繁访问的数据，这样你就不必一次又一次地访问数据库。

但开发者面临着一个两难选择：

我应该使用内存缓存（快速且简单）还是分布式缓存（可扩展且可共享）？

让我们通过一个实时示例来分析。

## 外卖示例

我们需要处理两种类型的数据：

1. 餐厅菜单 → 很少更改但被数百万用户访问。
2. 用户会话（购物车、订单跟踪）→ 动态的，需要在服务器之间共享。

## 内存缓存

内存缓存直接将数据存储在 Web/应用服务器的 RAM 中。

当请求到达时：

- 应用首先检查本地内存。
- 如果找到（缓存命中）→ 立即返回。
- 如果未找到（缓存未命中）→ 查询数据库 → 保存到内存 → 返回。

### 外卖示例：菜单

- 餐厅的菜单可以在服务器 RAM 中缓存 30 分钟。
- 在那段时间内的任何请求 → 从内存中提供服务，无需接触数据库。

```csharp
public class MenuService
{
    private readonly IMemoryCache _cache;

    public MenuService(IMemoryCache cache) => _cache = cache;

    public string GetMenu(string restaurantId)
    {
        return _cache.GetOrCreate(restaurantId, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
            return $"Menu for {restaurantId} loaded from DB";
        });
    }
}
```

优点

- 闪电般快速（无需网络调用）。
- 实现极其简单。
- 非常适合静态或很少变化的数据。

缺点

- 数据仅存在于单个服务器的内存中。
- 如果服务器重启 → 缓存将消失。
- 在多服务器设置中，每个服务器都有自己的数据副本→可能导致不一致。

## 分布式缓存

分布式缓存将数据存储在单独的缓存系统中，如 Redis、Memcached 或 NCache。

多个应用服务器连接到此缓存集群→所有服务器共享相同的数据。

当请求进来时：

- 应用查询分布式缓存（网络调用）。
- 如果缓存命中 → 返回。
- 如果未命中 → 从数据库加载 → 保存到分布式缓存中。

### 外卖示例：用户会话

- 用户登录并将食物添加到购物车。
- 无论他们的下一个请求是访问服务器 A 还是 B，购物车仍然可用（因为两者都从 Redis 读取）。

```csharp
public class SessionService
{
    private readonly IDistributedCache _cache;

    public SessionService(IDistributedCache cache) => _cache = cache;

    public async Task<string> GetUserCart(string userId)
    {
        var cart = await _cache.GetStringAsync(userId);
        if (cart == null)
        {
            cart = "Cart loaded from DB";
            await _cache.SetStringAsync(userId, cart,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }
        return cart;
    }
}
```

优点

- 跨服务器共享（非常适合负载均衡应用）。
- 即使单个应用服务器重启，缓存仍然保持。
- 处理大量数据。

缺点

- 稍慢（网络开销）。
- 需要基础设施设置（Redis 集群）。
- 需要监控和扩展。

## 其他现实场景

电商平台：

- 产品目录 → 内存中
- 用户购物车 → 分布式

银行应用：

- 汇率 → 内存中
- 用户交易和会话 → 分布式

媒体流：

- 热门电影列表 → 内存中
- 观看历史与用户资料 → 分布式

## 故障场景

内存缓存

- 服务器崩溃 = 缓存被清除。
- 如果进行水平扩展，用户可能会看到过时或缺失的数据，这取决于他们访问的是哪台服务器。

分布式缓存

- 如果 Redis 集群故障→缓存不可用（系统可能会回退到数据库，导致负载过重）。
- 网络问题可能会引入延迟。

这就是为什么生产环境通常部署 Redis 时会同时使用复制和持久化来确保可靠性。

## 最佳实践

混合方法 → 同时使用两种方式：

- 菜单/配置 → 内存存储
- 会话/购物车 → 分布式

设置过期策略

- 绝对时间（固定时间后清除）
- 滑动时间（在时间窗口内未使用时清除）

只缓存重要的内容

- 不要缓存所有内容。只缓存热点数据（高读取、低变更）。

回退策略

- 始终定义缓存为空时的处理方案（加载数据库，避免崩溃）。

## 总结

- 对小型、静态、本地化数据（菜单、功能标志、汇率）使用内存缓存。
- 对用户特定、动态、共享数据（会话、购物车、分析数据）使用分布式缓存。
- 实际系统会结合两者：使用内存缓存追求速度，使用分布式缓存实现扩展。

可以这样理解：

- 内存缓存 = 你的办公桌抽屉 → 极快，但只有你能使用。
- 分布式缓存 = 办公室文件柜 → 稍慢一些，但可共享且可靠。

今天就到这里。希望这些内容对你有帮助。
