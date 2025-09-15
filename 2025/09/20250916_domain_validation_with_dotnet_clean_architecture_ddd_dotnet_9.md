# [如何在 .NET 进行领域验证](https://medium.com/@michaelmaurice410/domain-validation-with-net-clean-architecture-ddd-net-9-0d80da162192)

在整洁架构和领域驱动设计（DDD）的世界中，验证应该放在两个地方：

1. 领域验证：确保实体和值对象始终根据业务规则保持有效状态。
2. 应用验证：在映射到领域对象之前验证客户端输入（命令/DTO）。

今天我将向您展示如何在 .NET 9 的整洁架构解决方案中实现健壮的领域验证。

## 领域验证原则

封装业务不变表征

- 实体和值对象必须在其构造函数或工厂方法中强制执行不变表征。
- 不应当展示无效状态。

使用卫语句（Guard Clauses）或规约模式（Specification Pattern）

- 卫语句将前置条件检查集中化。
- 规约模式表达复杂规则并且可以复用。

返回明确的结果

- 与其抛出通用异常，不如返回一个 Result\<T\> 或抛出领域特定的异常以表明更明确的意图。

避免贫血模型

- 实体应该包含行为和验证，而不仅仅是数据。

## 用于简单不变表征的卫语句

创建一个静态 Guard 辅助方法来强制执行常见检查：

```csharp
// Domain/Common/Guard.cs
namespace Domain.Common;
public static class Guard
{
    public static void AgainstNullOrEmpty(string value, string parameterName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainValidationException($"{parameterName} cannot be empty");
    }
    public static void AgainstNegative(decimal value, string parameterName)
    {
        if (value < 0)
            throw new DomainValidationException($"{parameterName} cannot be negative");
    }
    public static void AgainstZeroOrNegative(int value, string parameterName)
    {
        if (value <= 0)
            throw new DomainValidationException($"{parameterName} must be greater than zero");
    }
}
```

在你的实体或值对象构造函数中使用卫语句：

```csharp
// Domain/ValueObjects/Money.cs
namespace Domain.ValueObjects;
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
    public static Money Create(decimal amount, string currency)
    {
        Guard.AgainstNegative(amount, nameof(amount));
        Guard.AgainstNullOrEmpty(currency, nameof(currency));
        return new Money(amount, currency);
    }
}
```

## 工厂方法和私有构造函数

使用静态工厂防止无效实体的创建：

```csharp
// Domain/Entities/Order.cs
using Domain.Common;
using Domain.ValueObjects;
public class Order : Entity
{
    // 属性
    public DateTime OrderDate { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items;
    private readonly List<OrderItem> _items = [];
    private Order() { }  // EF Core
    public static Order Create(CustomerId customerId, DateTime orderDate)
    {
        if (orderDate > DateTime.UtcNow)
            throw new DomainValidationException("Order date cannot be in the future");
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            OrderDate = orderDate,
            Status = OrderStatus.Pending
        };
        return order;
    }
}
```

## 用于复杂规则的规约模式

当规则涉及多个实体或横切逻辑时，请使用规约模式：

```csharp
// Domain/Specifications/ISpecification.cs
namespace Domain.Specifications;
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    string ErrorMessage { get; }
}
// Domain/Specifications/OrderMustHaveItemsSpec.cs
public class OrderMustHaveItemsSpec : ISpecification<Order>
{
    public string ErrorMessage => "Order must contain at least one item";
    public bool IsSatisfiedBy(Order order) => order.Items.Any();
}
// Domain/Entities/Order.cs
public Result<Order> Confirm()
{
    var spec = new OrderMustHaveItemsSpec();
    if (!spec.IsSatisfiedBy(this))
        return Result.Failure<Order>(spec.ErrorMessage);
    Status = OrderStatus.Confirmed;
    return Result.Success(this);
}
```

## 用于明确结果的结果单子

使用 Result\<T\> 类型来返回成功或失败，而不使用异常：

```csharp
// Domain/Common/Result.cs
namespace Domain.Common;
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public string? Error { get; }
    protected Result(bool success, T? value, string? error)
    {
        IsSuccess = success;
        Value = value;
        Error = error;
    }
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

工厂返回 Result\<Order\> :

```csharp
public static Result<Order> Create(CustomerId id, DateTime date)
{
    if (date > DateTime.UtcNow)
        return Result.Failure<Order>("Order date cannot be in the future");
    
    var order = new Order { /*...*/ };
    return Result.Success(order);
}
```

## 应用验证与领域验证

- 应用层：在映射到领域模型之前，使用 FluentValidation 验证传入的命令/DTO。
- 领域层：使用卫语句、工厂方法和规约来强制执行业务不变表征。

FluentValidation 示例（应用层）

```csharp
// Application/Orders/Commands/CreateOrder/CreateOrderCommandValidator.cs
using FluentValidation;
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty();
        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item");
        RuleForEach(x => x.Items)
            .ChildRules(items =>
            {
                items.RuleFor(i => i.Quantity)
                     .GreaterThan(0);
                items.RuleFor(i => i.Price)
                     .GreaterThan(0);
            });
    }
}
```

## 测试领域验证

为领域不变表征编写单元测试：

```csharp
// Tests/Domain/OrderTests.cs
using Domain.Entities;
using Domain.Common;
using FluentAssertions;
using Xunit;
public class OrderTests
{
    [Fact]
    public void Create_OrderWithFutureDate_ShouldFail()
    {
        // Act
        var result = Order.Create(new CustomerId(Guid.NewGuid()), DateTime.UtcNow.AddDays(1));
        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be("Order date cannot be in the future");
    }
    [Fact]
    public void Confirm_OrderWithoutItems_ShouldFailSpecification()
    {
        var order = Order.Create(new CustomerId(Guid.NewGuid()), DateTime.UtcNow).Value;
        
        // Act
        var result = order.Confirm();
        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be("Order must contain at least one item");
    }
}
```

## 最佳实践

- 在领域中强制执行不变表征，以保持业务规则的集中化。
- 使用卫语句进行简单检查；使用规约处理复杂规则。
- 对于预期的验证失败返回 Result\<T\> ；将异常保留给真正异常的情况。
- 保持你的领域层不受应用程序关注点影响（不使用 FluentValidation 或数据注解）。
- 在映射到领域之前，在应用程序层验证命令/DTO。
- 为领域不变表征编写单元测试，确保它们在代码演化过程中保持有效。

## 结论

通过结合卫语句、工厂方法、规约模式和 Result\<T\> 模式，您可以在 .NET 9 中构建一个与清洁架构和领域驱动设计相符的健壮领域验证策略。这种方法确保您的领域模型保持一致性、可测试性，并且独立于外部框架，同时应用程序级验证处理客户端输入问题。最终得到的是一个可维护、可扩展的系统，业务规则在它们所属的位置——领域核心——得到强制执行。

今天就到这里。期待与你再次相见。
