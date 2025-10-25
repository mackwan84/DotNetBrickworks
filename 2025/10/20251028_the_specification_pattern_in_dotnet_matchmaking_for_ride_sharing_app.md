# [.NET 中的规范模式 — 拼车应用的匹配系统](https://medium.com/@dotnetfullstackdev/the-specification-pattern-in-net-matchmaking-for-ride-sharing-app-a4bd6ac29a9a)

是否曾经构建过一个规则不断变化的功能？硬编码的 if 丛林很快就会变成 bug。规范模式为你提供可重用、可组合的规则，你可以在运行时读取、测试和组合这些规则。

我们将使用一个真实场景：在拼车应用中将司机与乘车请求进行匹配。规则包括距离远近、座位数、司机评分、车型、高峰区域、合规检查等。明天产品要添加"仅限电动车、机场接送"功能？只需添加一个规范并组合它——无需重写。

## 什么是规范？

规范是一个回答"实体 X 是否满足规则 R？"的对象

你还可以将规范与 And/Or/Not 组合以形成更丰富的规则。使用 EF Core 时，规范可以暴露一个表达式用于服务器端转换。

## 最小构建块

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
    Expression<Func<T, bool>> ToExpression();
}

public abstract class Specification<T> : ISpecification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T candidate)
        => ToExpression().Compile().Invoke(candidate);

    public Specification<T> And(Specification<T> other)
        => new AndSpec<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpec<T>(this, other);

    public Specification<T> Not()
        => new NotSpec<T>(this);
}

sealed class AndSpec<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public AndSpec(Specification<T> left, Specification<T> right) 
        => (_left, _right) = (left, right);

    public override Expression<Func<T, bool>> ToExpression()
    {
        var left = _left.ToExpression();
        var right = _right.ToExpression();
        return left.AndAlso(right);
    }
}

sealed class OrSpec<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public OrSpec(Specification<T> left, Specification<T> right) 
        => (_left, _right) = (left, right);

    public override Expression<Func<T, bool>> ToExpression()
    {
        var left = _left.ToExpression();
        var right = _right.ToExpression();
        return left.OrElse(right);
    }
}

sealed class NotSpec<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpec(Specification<T> inner) => _inner = inner;

    public override Expression<Func<T, bool>> ToExpression()
    {
        var expr = _inner.ToExpression();
        return expr.Not();
    }
}

public static class PredicateBuilder
{
    public static Expression<Func<T, bool>> AndAlso<T>(
        this Expression<Func<T, bool>> left,
        Expression<Func<T, bool>> right)
    {
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(left, param),
            Expression.Invoke(right, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }

    public static Expression<Func<T, bool>> OrElse<T>(
        this Expression<Func<T, bool>> left,
        Expression<Func<T, bool>> right)
    {
        var param = Expression.Parameter(typeof(T));
        var body = Expression.OrElse(
            Expression.Invoke(left, param),
            Expression.Invoke(right, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }

    public static Expression<Func<T, bool>> Not<T>(
        this Expression<Func<T, bool>> expr)
    {
        var param = Expression.Parameter(typeof(T));
        var body = Expression.Not(Expression.Invoke(expr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}
```

## 领域模型

```csharp
public class Driver
{
    public int Id { get; set; }
    public GeoPoint Location { get; set; } = default!;
    public int SeatsAvailable { get; set; }
    public decimal Rating { get; set; }
    public string VehicleType { get; set; } = "";
    public bool IsOnline { get; set; }
    public bool BackgroundCheckPassed { get; set; }
    public bool AirportPermit { get; set; }
    public bool IsSurgeSuppressed { get; set; }
}

public record RideRequest(
    GeoPoint Pickup,
    int RequiredSeats,
    string PreferredVehicleType,
    bool AirportPickup);

public record GeoPoint(double Lat, double Lng);
```

## 具体规范

### 临近度

```csharp
public sealed class WithinRadiusSpec : Specification<Driver>
{
    private readonly GeoPoint _center;
    private readonly double _km;

    public WithinRadiusSpec(GeoPoint center, double km) => (_center, _km) = (center, km);

    public override Expression<Func<Driver, bool>> ToExpression()
        => d => DistanceKm(d.Location, _center) <= _km;

    private static double DistanceKm(GeoPoint a, GeoPoint b)
    {
        double dx = (a.Lat - b.Lat) * 111.32;
        double dy = (a.Lng - b.Lng) * 40075 * Math.Cos((a.Lat + b.Lat) * Math.PI/360) / 360;
        return Math.Sqrt(dx*dx + dy*dy);
    }
}
```

### 座位数

```csharp
public sealed class HasSeatsSpec : Specification<Driver>
{
    private readonly int _required;
    public HasSeatsSpec(int required) => _required = required;

    public override Expression<Func<Driver, bool>> ToExpression()
        => d => d.SeatsAvailable >= _required;
}
```

### 评分和在线状态

```csharp
public sealed class RatingAtLeastSpec : Specification<Driver>
{
    private readonly decimal _min;
    public RatingAtLeastSpec(decimal min) => _min = min;

    public override Expression<Func<Driver, bool>> ToExpression()
        => d => d.Rating >= _min;
}

public sealed class OnlineSpec : Specification<Driver>
{
    public override Expression<Func<Driver, bool>> ToExpression()
        => d => d.IsOnline;
}
```

### 车型偏好

```csharp
public sealed class VehicleTypeSpec : Specification<Driver>
{
    private readonly string _type; // "Any" allowed
    public VehicleTypeSpec(string type) => _type = type;

    public override Expression<Func<Driver, bool>> ToExpression()
        => d => _type == "Any" || d.VehicleType == _type;
}

public sealed class BackgroundClearedSpec : Specification<Driver>
{
    public override Expression<Func<Driver, bool>> ToExpression()
        => d => d.BackgroundCheckPassed;
}

public sealed class AirportPermitSpec : Specification<Driver>
{
    public override Expression<Func<Driver, bool>> ToExpression()
        => d => d.AirportPermit;
}
```

### 策略切换（例如，阻止被抑制高峰加价的司机）

```csharp
public sealed class NotSurgeSuppressedSpec : Specification<Driver>
{
    public override Expression<Func<Driver, bool>> ToExpression()
        => d => !d.IsSurgeSuppressed;
}
```

## 组合规范以匹配司机

业务需求：

- 查找 3 公里范围内的司机
- 在线，评分≥4.7，座位数≥请求数量
- 尊重车辆偏好
- 如果是机场接单，需要机场许可证
- 排除被抑制加价的司机

```csharp
public static Specification<Driver> BuildMatchSpec(RideRequest req)
{
    Specification<Driver> baseSpec =
        new WithinRadiusSpec(req.Pickup, 3.0)
        .And(new OnlineSpec())
        .And(new RatingAtLeastSpec(4.7m))
        .And(new HasSeatsSpec(req.RequiredSeats))
        .And(new VehicleTypeSpec(req.PreferredVehicleType))
        .And(new NotSurgeSuppressedSpec())
        .And(new BackgroundClearedSpec());

    if (req.AirportPickup)
        baseSpec = baseSpec.And(new AirportPermitSpec());

    return baseSpec;
}
```

结合 EF Core 查询：

```csharp
public async Task<List<Driver>> FindCandidatesAsync(
    RideRequest req, DbContext db, CancellationToken ct = default)
{
    var spec = BuildMatchSpec(req);
    return await db.Set<Driver>()
                   .Where(spec.ToExpression())
                   .OrderBy(d => d.Rating descending)
                   .ToListAsync(ct);
}
```

尽在内存中过滤：

```csharp
var allDrivers = GetFromCache();
var spec = BuildMatchSpec(req);
var candidates = allDrivers.Where(spec.IsSatisfiedBy).ToList();
```

同一规则对象既支持数据库查询，也支持内存中的决策。

## 为什么不直接使用 LINQ？

LINQ 确实可以表达单个查询。但这里的优势在于可重用性和组合性：

- "符合机场条件的司机"成为一条命名规则，可用于匹配、支付结算和审计。
- 产品切换功能（如"校园内仅电动车接单"）变成了可插拔的规格，而不是到处添加新的 if 。
- 你可以单独对每个规则进行单元测试。

## 如何测试

```csharp
[Fact]
public void SeatsSpec_Rejects_TooFewSeats()
{
    var spec = new HasSeatsSpec(required: 3);
    var driver = new Driver { SeatsAvailable = 2 };
    Assert.False(spec.IsSatisfiedBy(driver));
}
```

## 扩展运用

- 验证规范（领域不变量）：例如， IsReadyForPayoutSpec(driver)
- 查询规范（过滤）：我们上面构建的内容
- 投影规范：将谓词与 Select 配对，以集中化 DTO 塑形
- 缓存+规范：为热点规则缓存 ToExpression().Compile()

## 常见陷阱与技巧

- EF Core 转换：在 ToExpression 中只使用表达式树友好的代码。将不可翻译的数学运算推送到数据库函数或在内存中处理。
- 过度规范：不要创建混合规则和副作用的规范。保持它们的纯粹性。
- 命名：将规范视为通用语言： AirportPermitSpec 、 EVOnlySpec 、 HighDemandZoneSpec 。
- 性能：尽可能在启动时组合；为静态规则重用实例。

## 总结

- 规范模式将混乱的条件语句转化为干净、可组合、可测试的规则对象。
- 在我们的网约车示例中，我们为距离、容量、评分、状态、合规性和政策构建了规范——然后将它们组合起来以匹配司机。
- 同一规范支持 EF 查询和内存检查，因此你的规则集中在一个地方。

今天就到这里。希望这些内容对你有帮助。
