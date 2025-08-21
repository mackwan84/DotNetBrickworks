# [.NET 中的值对象（领域驱动设计基础）](https://www.milanjovanovic.tech/blog/value-objects-in-dotnet-ddd-fundamentals)

值对象是领域驱动设计的构建块之一。DDD 是一种用于解决复杂领域问题的软件开发方法。

值对象封装了一组原始值和相关不变量。值对象的几个例子包括金额和日期范围对象。金额由数量和货币组成。日期范围由开始日期和结束日期组成。

今天，我将向你展示实现值对象的一些最佳实践。

## 什么是值对象？

让我们从《领域驱动设计》一书中的定义开始：

> 一个表示领域中描述性方面且没有概念身份的对象称为值对象。值对象的实例化是为了表示我们只关心它们是什么，而不关心它们是谁或哪个的设计元素。
> Eric Evans

值对象与实体不同——它们没有身份的概念。它们封装了领域中的基本类型，并解决了基本类型痴迷问题。

值对象有两个主要特性：

- 它们是不可变的
- 它们没有身份标识

值对象的另一个特性是结构相等性。如果两个值对象的值相同，那么它们就是相等的。这个特性在实际应用中是最不重要的。然而，在某些情况下，你可能只需要部分值来决定相等性。

## 实现值对象

值对象最重要的特性是不可变性。值对象的值一旦创建就不能更改。如果要更改单个值，需要替换整个值对象。

这是一个使用原始值表示地址以及预订开始和结束日期的 Booking 实体。

```csharp
public class Booking
{
    public string Street { get; init; }
    public string City { get; init; }
    public string State { get; init; }
    public string Country { get; init; }
    public string ZipCode { get; init; }

    public DateOnly StartDate { get; init; }
    public DateOnly EndDate { get; init; }
}
```

你可以用 Address 和 DateRange 值对象来替换这些原始值。

```csharp
public class Booking
{
    public Address Address { get; init; }

    public DateRange Period { get; init; }
}
```

但是，你如何实现值对象呢？

## C# 记录

你可以使用 C#记录来表示值对象。记录在设计上是不可变的，并且它们具有结构相等性。我们的值对象需要这两种特性。

例如，你可以使用带有主构造函数的 record 来表示一个 Address 值对象。这种方法的优点是简洁性。

```csharp
public record Address(
    string Street,
    string City,
    string State,
    string Country,
    string ZipCode);
```

然而，在定义私有构造函数时，你会失去这一优势。当你想要在创建值对象时强制不变性时，就会出现这种情况。使用记录的另一个问题是使用 with 表达式来避免值对象不变性。

```csharp
public record Address
{
    private Address(
        string street,
        string city,
        string state,
        string country,
        string zipCode)
    {
        Street = street;
        City = city;
        State = state;
        Country = country;
        ZipCode = zipCode;
    }

    public string Street { get; init; }
    public string City { get; init; }
    public string State { get; init; }
    public string Country { get; init; }
    public string ZipCode { get; init; }

    public static Result<Address> Create(
        string street,
        string city,
        string state,
        string country,
        string zipCode)
    {
        // 检查地址是否有效

        return new Address(street, city, state, country, zipCode);
    }
}
```

## 基类

实现值对象的另一种方法是使用 ValueObject 基类。该基类通过 GetAtomicValues 抽象方法处理结构相等性。 ValueObject 实现必须实现此方法并定义相等性组件。

使用基类的优势在于它是显式的。可以清楚地知道领域中的哪些类表示值对象。另一个优势是能够控制相等性组件。

这是我在项目中使用的 ValueObject 基类：

```csharp
public abstract class ValueObject : IEquatable<ValueObject>
{
    public static bool operator ==(ValueObject? a, ValueObject? b)
    {
        if (a is null && b is null)
        {
            return true;
        }

        if (a is null || b is null)
        {
            return false;
        }

        return a.Equals(b);
    }

    public static bool operator !=(ValueObject? a, ValueObject? b) =>
        !(a == b);

    public virtual bool Equals(ValueObject? other) =>
        other is not null && ValuesAreEqual(other);

    public override bool Equals(object? obj) =>
        obj is ValueObject valueObject && ValuesAreEqual(valueObject);

    public override int GetHashCode() =>
        GetAtomicValues().Aggregate(
            default(int),
            (hashcode, value) =>
                HashCode.Combine(hashcode, value.GetHashCode()));

    protected abstract IEnumerable<object> GetAtomicValues();

    private bool ValuesAreEqual(ValueObject valueObject) =>
        GetAtomicValues().SequenceEqual(valueObject.GetAtomicValues());
}
```

Address 值对象的实现将如下所示：

```csharp
public sealed class Address : ValueObject
{
    public string Street { get; init; }
    public string City { get; init; }
    public string State { get; init; }
    public string Country { get; init; }
    public string ZipCode { get; init; }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return Street;
        yield return City;
        yield return State;
        yield return Country;
        yield return ZipCode;
    }
}
```

## 何时使用值对象？

我使用值对象来解决基本类型痴迷问题并封装领域不变量。封装是任何领域模型的重要方面。你不应该能够创建一个处于无效状态的值对象。

值对象也为你提供了类型安全。看看这个方法签名：

```csharp
public interface IPricingService
{
    decimal Calculate(Apartment apartment, DateOnly start, DateOnly end);
}
```

然后，将其与我们添加了值对象的方法签名进行比较。你可以看到使用值对象的 IPricingService 要明确得多。你还获得了类型安全的好处。在编译代码时，值对象减少了错误潜入的机会。

```csharp
public interface IPricingService
{
    PricingDetails Calculate(Apartment apartment, DateRange period);
}
```

以下是你在决定是否需要值对象时应考虑的更多事项：

- **不变量的复杂性** - 如果要强制执行复杂的不变量，请考虑使用值对象
- **原始值的数量** - 当封装许多原始值时，值对象才有意义
- **重复次数** - 如果你只需要在代码中的少数几个地方强制不变性，那么不使用值对象也可以管理

## 使用 EF Core 持久化值对象

值对象是领域实体的一部分，你需要将它们保存在数据库中。

我将向你展示如何使用 EF 拥有的类型和复杂类型来持久化值对象。

### Owned Types 拥有类型

在配置实体时，可以通过调用 OwnsOne 方法来配置自有类型。这会告诉 EF 将 Address 和 Price 值对象持久化到与 Apartment 实体相同的表中。值对象在 apartments 表中用额外的列表示。

```csharp
public void Configure(EntityTypeBuilder<Apartment> builder)
{
    builder.ToTable("apartments");

    builder.OwnsOne(property => property.Address);

    builder.OwnsOne(property => property.Price, priceBuilder =>
    {
        priceBuilder.Property(money => money.Currency)
            .HasConversion(
                currency => currency.Code,
                code => Currency.FromCode(code));
    });
}
```

关于拥有类型的几点补充说明：

- 自有类型具有隐藏的键值
- 不支持可选（可空）的拥有类型
- 支持使用 OwnsMany 的拥有集合
- 表拆分允许你单独持久化拥有类型

### Complex Types 复杂类型

复杂类型是 .NET 8 中提供的一项新 EF 功能。它们不由键值标识或跟踪。复杂类型必须是实体类型的一部分。

使用 EF 时，复杂类型更适合表示值对象。

以下是如何将 Address 值对象配置为复杂类型：

```csharp
public void Configure(EntityTypeBuilder<Apartment> builder)
{
    builder.ToTable("apartments");

    builder.ComplexProperty(property => property.Address);
}
```

复杂类型的几个限制：

- 不支持集合
- 不支持可空值

## 总结

值对象有助于设计丰富的领域模型。你可以使用它们来解决原始类型痴迷问题并封装领域不变量。值对象可以通过防止无效领域对象的实例化来减少错误。

你可以使用 record 或 ValueObject 基类来表示值对象。这应该取决于你的具体需求和领域的复杂性。我默认使用记录，除非我需要 ValueObject 基类的某些特性。例如，当你想要控制相等性组件时，基类非常实用。

希望这对你有所帮助。如果你有任何问题或想法，请在评论中告诉我。

今天就到这里。期待与你再次相见。
