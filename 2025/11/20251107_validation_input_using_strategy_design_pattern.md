# [使用策略设计模式的验证输入](https://medium.com/dev-genius/validation-input-using-strategy-design-pattern-dc445b56a11a)

我们将应用策略设计模式来验证用户输入

## 背景

我们正在设计用户注册模块，并且需要设计一个系统来根据用户类型验证输入请求。

让我们讨论一个糟糕的代码示例

```csharp
public class RegistrationService
{
    public List<string> Validate(UserRegistrationRequest request)
    {
        var errors = new List<string>();

        if (request.UserType == "Individual")
        {
            if (string.IsNullOrEmpty(request.FullName))
                errors.Add("Full name is required.");
            if (request.Age < 18)
                errors.Add("User must be at least 18 years old.");
        }
        else if (request.UserType == "Business")
        {
            if (string.IsNullOrEmpty(request.CompanyName))
                errors.Add("Company name is required.");
            if (string.IsNullOrEmpty(request.RegistrationNumber))
                errors.Add("Registration number is required.");
        }

        return errors;
    }
}
```

问题所在

每次需要添加新类型的用户时，我们需要添加另一个 if 条件

- 难以维护：随着添加更多用户类型，逻辑变得混乱。
- 不可重用：逻辑紧密耦合在一个地方。
- 对扩展封闭：添加新类型（例如政府用户）需要修改相同的代码。

解决方案

我们有多种解决方案来处理这种情况，但我们将专注于策略设计模式。

根据策略设计模式，我们可以在运行时注入逻辑。

这意味着我们可以将逻辑创建到类中，并在运行时注入或执行这些逻辑。

## 策略设计模式

### 第一步

定义要验证的内容 -> 我们将要验证用户注册请求

```csharp
public interface IValidationStrategy
{
    List<string> Validate(UserRegistrationRequest request);
}

public class UserRegistrationRequest
{
    public string UserType { get; set; } // "Individual", "Business"
    public string FullName { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }
    public string CompanyName { get; set; }
    public string RegistrationNumber { get; set; }
}
```

### 第二步

根据用户类型为上述接口定义具体实现

```csharp
public class IndividualUser
{
    public class IndividualUserValidationStrategy : IValidationStrategy
    {
        public List<string> Validate(UserRegistrationRequest request)
        {
            var errors = new List<string>();

            if (string.IsNullOrWhiteSpace(request.FullName))
                errors.Add("Full name is required.");
            if (request.Age < 18)
                errors.Add("User must be at least 18 years old.");
            if (!request.Email.Contains("@"))
                errors.Add("Valid email is required.");

            return errors;
        }
    }
}
```

### 第三步

定义验证存储库

它包含所有用户类型的验证逻辑，并根据用户输入返回相应的策略

它是一个类，充当中央协调器，根据 UserType 选择合适的验证策略。

```csharp
public class ValidationContext
{
    private readonly Dictionary<string, IValidationStrategy> _strategies;

    public ValidationContext()
    {
        _strategies = new Dictionary<string, IValidationStrategy>
        {
            { "Individual", new IndividualUserValidationStrategy() },
            { "Business", new BusinessUserValidationStrategy() }
        };
    }

    public List<string> Validate(UserRegistrationRequest request)
    {
        if (_strategies.TryGetValue(request.UserType, out var strategy))
        {
            return strategy.Validate(request);
        }

        return new List<string> { "Unsupported user type." };
    }
}
```

解释说明

- 维护策略列表 -> 它保存一个从 UserType 到 IValidationStrategy 的映射
- 在运行时选择策略 -> 基于 UserType ，它从字典中选择正确的策略
- 委托验证 -> 它在所选策略上调用 Validate()

### 第四步

执行

```csharp
[HttpPost("register")]
public IActionResult Register([FromBody] UserRegistrationRequest request)
{
    var errors = _validationContext.Validate(request);
    if (errors.Any())
    {
        return BadRequest(new { Errors = errors });
    }

    return Ok("Registration successful!");
}
```

示例 JSON

```json
{
    "userType": "Individual",
    "fullName": "devesh",
    "age": 7,
    "email": "devesh.akgec@gmail.com",
    "companyName": "Dotnet LABS",
    "registrationNumber": "9891"
}
```

我们添加了年龄应大于 18 岁的逻辑...因此收到此验证消息。

## 总结

使用策略模式，我们获得以下好处

- 解耦、可维护的类
- 代码可扩展 -> 易于添加新的用户类型
- 遵循开闭原则
- 清晰的关注点分离

今天就到这里。希望这些内容对你有帮助。
