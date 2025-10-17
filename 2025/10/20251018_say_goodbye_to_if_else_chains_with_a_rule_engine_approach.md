# [使用规则引擎方法告别 If-Else 链](https://medium.com/@devesh.akgec/say-goodbye-to-if-else-chains-with-a-rule-engine-approach-cc0b8b0695f7)

> 通过使用这种方法，我们可以使代码更具可扩展性和可管理性，并消除冗长的 if-else 条件链

今天，我们将学习如何巧妙地避免冗长的 if-else 链。

## 理解问题

在软件开发中，if-else 语句通常用于根据不同条件执行不同的代码块。然而，当条件变得复杂且数量众多时，if-else 链会变得冗长且难以维护。这不仅影响代码的可读性，还增加了出错的风险。

假设我们对客户信息的定义如下

```csharp
internal class Customer
 {
     public string CustomerType { get; set; }
     public int Age { get; set; }
     public decimal TotalPurchases { get; set; }
 }
 ```

 现有的业务逻辑：

 ```csharp
public void ApplyBusinessLogic(Customer objCustomer)
{
    if (objCustomer.CustomerType == "Gold")
    {
        CallGoldLogic();
    }

    if (objCustomer.CustomerType == "Premeium")
    {
        CallPremiumLogic();
    }

    if (objCustomer.Age > 60)
    {
        CallDiscountLogic();
    }

    if (objCustomer.TotalPurchases > 6)
    {
        CallPurchaseLogic();
    }
    else
    {
        CallRegularLogic();
    }
}
```

在这里我们可以看到有很多 If 和 else。这很难管理和扩展

我们在这里有以下问题

- 违反单一职责原则：将所有内容添加到同一个类中违反了单一职责原则
- 单元测试用例：我们必须多次传递客户对象。为这样长的函数编写单元测试用例是一项繁琐的任务。
- 代码复杂性
- 可扩展性：我们需要向现有函数添加代码，这需要再次进行端到端测试。在这种情况下，代码不具备可扩展性

## 解决方案

我们有多种解决方案来处理这种情况，但今天，我将重点介绍基于规则引擎的方法。

首先尝试查看类中是否存在一些问题，我按以下方式修改了类。

修改后的客户类

```csharp
internal class Customer
{
    public bool IsGold { get; set; }
    public bool IsPremium { get; set; }
    public int Age { get; set; }
    public decimal TotalPurchases { get; set; }
}
```

改动：

- 我将 CustomerType 更改为 IsGold 和 IsPremium 布尔属性。这使得代码更具可读性，并且在检查客户类型时更容易理解。
- 其他属性保持不变。

### 定义规则引擎类

通过定义一个规则类来表示每个规则的条件和操作，我们可以更好地组织代码。

```csharp
public class CustomerRule
{
    public string Name { get; set; }
    public Func<Customer, bool> Condition { get; set; }
    public Action<Customer> Action { get; set; }
}
```

添加规则

```csharp
var rules = new List<CustomerRule>
{
    new CustomerRule
    {
        Name = "Gold Discount Rule",
        Condition = c => c.IsGold,
        Action = c => Console.WriteLine("Applied 20% discount for Gold member.")
    },
    new CustomerRule
    {
        Name = "Premium Senior Gift Rule",
        Condition = c => c.IsPremium && c.Age > 60,
        Action = c => Console.WriteLine("Sent free gift to Premium senior.")
    },
    new CustomerRule
    {
        Name = "VIP Club Invitation Rule",
        Condition = c => c.TotalPurchases > 10_000,
        Action = c => Console.WriteLine("Invited to VIP Club.")
    },
    new CustomerRule
    {
        Name = "Youth Promo Notification Rule",
        Condition = c => c.Age < 25,
        Action = c => Console.WriteLine("Sent youth promotion.")
    }
};
```

处理与执行

```csharp
public void ApplyRules(Customer customer, List<CustomerRule> rules)
{
    foreach (var rule in rules)
    {
        if (rule.Condition(customer))
        {
            Console.WriteLine($"[Rule Matched: {rule.Name}]");
            rule.Action(customer);
        }
    }
}
```

运行代码

```csharp
var customer = new Customer
{
    IsGold = true,
    IsPremium = true,
    Age = 65,
    TotalPurchases = 12000
};

ApplyRules(customer, rules);
```

输出

```bash
[Rule Matched: Gold Discount Rule]
Applied 20% discount for Gold member.
```

## 总结

我们已经探讨了如何使用 Action<> 和 Func<> 来替换长的 If-else 链。我们通过应用了如上所述的规则引擎，可以为未来的需求扩展我们的代码。

希望今天的分享对你有帮助。

今天就到这里。期待与你再次相见。
