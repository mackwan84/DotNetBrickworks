# [.NET 中的策略模式与工厂模式——通过投资计划理解其区别](https://medium.com/@dotnetfullstackdev/strategy-pattern-vs-factory-pattern-in-net-understanding-the-difference-with-an-investment-plan-ebd707b83ddf)

在企业系统，尤其是金融领域工作时，你经常会遇到需要动态选择不同算法或创建逻辑的情况。这时，两种著名的设计模式就派上用场了：

- **策略模式（Strategy Pattern）**：用于在运行时选择不同的算法或行为。
- **工厂模式（Factory Pattern）**：用于创建对象，隐藏具体的实例化逻辑。

它们看起来相似，但承担着非常不同的职责。让我们深入探讨。

## 业务问题：投资计划

想象一下，你正在构建一个向客户提供不同投资计划的金融科技系统：

- 激进型计划 → 高风险，高回报
- 平衡型计划 → 中等风险，中等回报
- 保守型计划 → 低风险，低回报

出现两个常见需求：

- 策略需求 → 在运行时动态应用不同的计算算法（回报、风险评分）。
- 工厂需求 → 根据用户选择或外部输入创建正确的投资计划对象。

## 策略模式 — 关注行为

策略模式指出："定义一系列算法，将每个算法放入单独的类中，并使它们在运行时可以互换。"

### 步骤 1：定义策略接口

```csharp
public interface IInvestmentStrategy
{
    decimal CalculateReturn(decimal amount, int years);
}
```

### 步骤 2：具体策略

```csharp
public class AggressiveStrategy : IInvestmentStrategy
{
    public decimal CalculateReturn(decimal amount, int years)
        => amount * (decimal)Math.Pow(1.15, years); // 15% growth
}

public class BalancedStrategy : IInvestmentStrategy
{
    public decimal CalculateReturn(decimal amount, int years)
        => amount * (decimal)Math.Pow(1.08, years); // 8% growth
}

public class ConservativeStrategy : IInvestmentStrategy
{
    public decimal CalculateReturn(decimal amount, int years)
        => amount * (decimal)Math.Pow(1.04, years); // 4% growth
}
```

### 步骤 3：上下文类

```csharp
public class InvestmentContext
{
    private readonly IInvestmentStrategy _strategy;

    public InvestmentContext(IInvestmentStrategy strategy)
    {
        _strategy = strategy;
    }

    public decimal ExecuteStrategy(decimal amount, int years)
    {
        return _strategy.CalculateReturn(amount, years);
    }
}
```

### 使用方法

```csharp
var context = new InvestmentContext(new BalancedStrategy());
var returns = context.ExecuteStrategy(10000, 5);
Console.WriteLine($"Projected return: {returns:C}");
```

策略模式的核心是在运行时切换行为（算法）。

## 工厂模式 — 专注于对象创建

工厂模式指出："集中对象的创建逻辑，使客户端无需知道应该实例化哪个类。"

在这里，我们不直接创建 AggressiveStrategy 或 BalancedStrategy ，而是让工厂来决定。

工厂示例

```csharp
public static class InvestmentStrategyFactory
{
    public static IInvestmentStrategy CreateStrategy(string type) =>
        type.ToLower() switch
        {
            "aggressive" => new AggressiveStrategy(),
            "balanced"   => new BalancedStrategy(),
            "conservative" => new ConservativeStrategy(),
            _ => throw new ArgumentException("Invalid strategy type")
        };
}
```

### 使用方法

```csharp
var strategy = InvestmentStrategyFactory.CreateStrategy("aggressive");
var context = new InvestmentContext(strategy);

var returns = context.ExecuteStrategy(20000, 10);
Console.WriteLine($"Projected return: {returns:C}");
```

工厂模式是关于对象创建封装的。客户端只需向工厂请求，而不需要担心实例化具体类。

## 在实际系统中的组合使用

在实践中，你经常同时使用两者：

- 工厂决定实例化哪种策略（基于配置、用户选择或市场规则）。
- 策略执行行为（计算回报、风险分析）。

示例

```csharp
var chosenStrategy = InvestmentStrategyFactory.CreateStrategy("conservative");
var context = new InvestmentContext(chosenStrategy);

var result = context.ExecuteStrategy(50000, 7);
Console.WriteLine($"Conservative plan returns: {result:C}");
```

## 现实世界类比

- 策略 = 厨师的烹饪风格（辛辣、清淡、健康）。
- 工厂 = 餐厅菜单，为你选择分配哪位厨师。
- 结合起来 → 你点"均衡餐"（工厂选择厨师）→ 厨师以他的风格烹饪（策略）。

## 总结

- 策略 → 定义行为如何动态变化。
- 工厂 → 定义如何在不暴露 new 的情况下创建对象。
- 它们看起来相似，因为两者都涉及接口和多个实现，但它们的职责是不同的。
- 在实际项目（如投资计划）中，你通常会结合使用它们：工厂选择合适的策略，策略定义算法。
