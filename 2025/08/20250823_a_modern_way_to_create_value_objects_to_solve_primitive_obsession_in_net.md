# [在 .NET 中创建值对象以解决基本类型痴迷的现代方法](https://antondevtips.com/blog/a-modern-way-to-create-value-objects-to-solve-primitive-obsession-in-net)

**基本类型痴迷** Primitive obsession 是一种倾向于使用基本数据类型来表示更复杂概念的现象。它是一种常见的反模式，可能导致代码不清晰且系统难以维护。

今天，我将向你解释为什么原始痴迷会导致应用程序中的问题，以及如何使用 .NET 中的值对象来解决这个问题。

## 为什么基本类型痴迷是一个问题？

**基本类型痴迷**是指当基本数据类型（如 int、string 或 DateTime）被过度使用来表示领域中的复杂概念时发生的情况。这种做法可能导致代码不清晰、产生错误，以及难以维护和扩展系统。

为什么这会成为一个问题呢？

- **缺乏表达性**：像 int 或 string 这样的基本类型无法清晰表达它们所代表数据的含义。例如，一个字符串可以是电子邮件地址、用户名或 URL，但仅通过查看类型是无法区分的。
- **错误风险增加**：当你为不同的概念使用相同的原始类型时，很容易传递错误的数据。例如，对用户名和电子邮件地址都使用字符串可能导致在需要用户名的地方错误地传递电子邮件。
- **分散的验证逻辑**：基本类型的验证逻辑通常分散在整个代码库中。每次需要检查字符串是否为有效的电子邮件地址时，都需要编写验证逻辑，这可能导致重复和不一致。
- **代码演变困难**：随着应用程序的增长，需求可能会发生变化。如果你在各处都使用了基本类型，那么要以一致且不破坏性的方式进行这些更改就会变得困难。

## 基本类型痴迷的示例

让我们探索一个示例，其中某个 User 实体具有 Email 属性。在创建用户时，你可能会在创建 User 对象时实现验证。

你可能会争辩说，既然你的 WebAPI 请求已经有输入验证，为什么还需要这样的验证。但事实是，对输入请求的验证并不能保护你免受代码中出现的错误，比如在应用程序的其他组件中传递或映射错误的参数。这种验证在领域驱动设计中是一种流行的方式，你在创建对象时会建立一个最终的安全保障。

让我们来探索一个示例（简化了电子邮件验证，不适合生产环境使用）：

```csharp
public class User
{
    public string Email { get; set; }

    public User(string email)
    {
        if (string.IsNullOrWhiteSpace(email) || !email.Contains("@"))
        {
            throw new ArgumentException("Invalid email address", nameof(email));
        }
        
        Email = email;
    }
}
```

为什么这是一个问题？

- Email 属性只是一个字符串，所以它可以保存任何类型的字符串，而不仅仅是有效的电子邮件地址。
- 验证逻辑嵌入在构造函数中，使得在其他需要使用电子邮件的地方难以重用。
- 如果代码库的另一部分需要处理电子邮件，验证逻辑可能需要重复。

让我们来看另一个例子：

```csharp
public class User
{
    public string Email { get; }
    public string Username { get; }

    public User(string email, string username)
    {
        if (string.IsNullOrWhiteSpace(email))
        {
            throw new ArgumentException("Email is required", nameof(email));
        }
        
        if (string.IsNullOrWhiteSpace(username))
        {
            throw new ArgumentException("Username is required", nameof(username));
        }

        Email = email;
        Username = username;
    }
}
```

让我们尝试创建一个用户：

```csharp
var user = new User("anton@test.com", "anton");
var user = new User("anton", "anton@test.com");
```

这段代码编译并成功执行。你注意到问题了吗？

由于我们同时使用 string 类型来表示电子邮件和用户名 - 你可能会以错误的顺序传递参数。如果在构造对象时没有像我上面展示的那样进行验证，最终数据库中会出现错误的数据。这可能导致严重的问题。

解决这个问题的方法是使用**值对象**，它将相关数据和行为封装成一个单一且有意义的单元。

## 什么是值对象？

**值对象**表示领域中一个没有身份但由其属性定义的值。例如，一个 Address 可能是一个**值对象**，因为它是由其属性（街道、城市、邮政编码）定义的。

**值对象**是领域驱动设计（DDD）中的一个关键概念。

### 值对象的关键特征

- 不可变性：一旦创建，值对象就不能被更改。任何修改都会产生一个新实例。
- 相等性：值对象是基于其属性进行比较，而不是通过引用。
- 自验证：它们通过对属性强制执行约束来确保其状态始终有效。

### 使用值对象的好处

- **表达性**：代码变得更加易读和自解释。
- **封装**：业务规则被封装在值对象中，确保在所有使用它们的地方保持一致性。
- **减少错误**：通过限制基本类型的范围，你可以降低传递错误数据的风险。
- **可重用性**：值对象可以在应用程序的不同部分重复使用，促进 DRY（不要重复自己）原则的应用。

现在让我们来探讨在.NET 中创建值对象有哪些选项。

## 一个示例应用程序

今天我将向你展示如何为一个负责创建和更新客户、订单以及已订购产品发货的应用程序实现**值对象**。

此应用程序包含以下实体：

- 客户
- 订单、订单项
- 货物，货物项

我正在为我的实体使用领域驱动设计实践。让我们来探索一个 Customer 和 Shipment 实体，它们为所有属性使用基本类型：

```csharp
public class Customer
{
    public Guid Id { get; private set; }
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public string Email { get; private set; }
    public string PhoneNumber { get; private set; }
    public IReadOnlyList<Order> Orders => _orders.AsReadOnly();

    private readonly List<Order> _orders = [];

    private Customer() { }

    public static Customer Create(
        string firstName,
        string lastName,
        string email,
        string phoneNumber)
    {
        return new Customer
        {
            Id = Guid.NewGuid(),
            FirstName = firstName,
            LastName = lastName,
            Email = email,
            PhoneNumber = phoneNumber
        };
    }

    public void AddOrder(Order order)
    {
        _orders.Add(order);
    }
}
```

```csharp
public class Shipment
{
    private readonly List<ShipmentItem> _items = [];

    public Guid Id { get; private set; }

    public string Number { get; private set; }

    public Guid OrderId { get; private set; }

    public string Address { get; private set; }

    public string Carrier { get; private set; }

    public string ReceiverEmail { get; private set; }

    public ShipmentStatus Status { get; private set; }

    public IReadOnlyList<ShipmentItem> Items => _items.AsReadOnly();

    public DateTime CreatedAt { get; private set; }

    public DateTime? UpdatedAt { get; private set; }

    private Shipment()
    {
    }

    public static Shipment Create(
        string number,
        Guid orderId,
        string address,
        string carrier,
        string receiverEmail,
        List<ShipmentItem> items)
    {
        var shipment = new Shipment
        {
            Id = Guid.NewGuid(),
            Number = number,
            OrderId = orderId,
            Address = address,
            Carrier = carrier,
            ReceiverEmail = receiverEmail,
            Status = ShipmentStatus.Created,
            CreatedAt = DateTime.UtcNow
        };

        shipment.AddItems(items);

        return shipment;
    }

    public void AddItems(List<ShipmentItem> items)
    {
        _items.AddRange(items);
        UpdatedAt = DateTime.UtcNow;
    }

    public void AddItem(ShipmentItem item)
    {
        _items.Add(item);
        UpdatedAt = DateTime.UtcNow;
    }
}
```

这些实体痴迷于基本类型：货运编号、地址、电子邮件地址、电话号码等。让我们探讨如何用值对象替换这些基本类型。

## 在 .NET 中创建值对象

我可以列出以下几种创建值对象的最流行选项：

- 使用 ValueOf 库
- 使用 record 记录类型
- 使用 record struct 记录结构体

让我们更深入地探讨每个选项。

## 使用 ValueOf 创建值对象

ValueOf 是一个流行的库，它通过提供一个基类来处理大量样板代码，从而创建值对象。

首先，你需要安装这个包：

```bash
dotnet add package ValueOf
```

以下是如何使用 ValueOf 库创建值对象的方法：

```csharp
using ValueOf;

public class EmailAddress : ValueOf<string, EmailAddress>
{
    protected override void Validate()
    {
        if (string.IsNullOrWhiteSpace(Value) || !Value.Contains("@"))
        {
            throw new ArgumentException("Invalid email address.");
        }
    }
}

public class OrderNumber : ValueOf<string, OrderNumber>
{
    protected override void Validate()
    {
        if (string.IsNullOrWhiteSpace(Value))
        {
            throw new ArgumentException("Order number cannot be empty.");
        }
    }
}
```

你需要让你的值对象（EmailAddress 和 OrderNumber）类继承自一个基 ValueOf 类，并提供 2 个泛型类型：

- 一个下划线原始类型
- 一个值对象类型本身

ValueOf 支持 C# 元组，如果你需要将值对象表示为多个属性，例如，地址：

```csharp
using ValueOf;

public class Address : ValueOf<(string Street, string City, string Zip), Address>
{
    protected override void Validate()
    {
        if (string.IsNullOrWhiteSpace(Value.Street))
        {
            throw new ArgumentException("Street cannot be empty.");
        }
        
        if (string.IsNullOrWhiteSpace(Value.City))
        {
            throw new ArgumentException("City cannot be empty.");
        }
        
        if (string.IsNullOrWhiteSpace(Value.Zip))
        {
            throw new ArgumentException("Zip code cannot be empty.");
        }
    }
}
```

以下是创建这些值对象的方法：

```csharp
var email = EmailAddress.From("anton@test.com");
var orderNumber = OrderNumber.From("ORD-12345");
var address = Address.From(("123 Main St", "Springfield", "12345"));
```

你可以使用 Value 属性来检索隐藏在值对象内部的值：

```csharp
string emailValue = email.Value;
string orderNumberValue = orderNumber.Value;
(string street, string city, string zip) = address.Value;
```

## 使用记录类型创建值对象

有些开发者使用 ValueOf 库来实现值对象，甚至更多的人创建自己的实现。如果我告诉你 C# 记录已经包含了值对象所需的一切，你会怎么想？

记录类型是在 .NET 中创建值对象的一种非常现代的方式。

记录类型是不可变的引用类型，它们开箱即用地支持相等性比较。它们是基于属性进行比较的，而不是通过引用。它还开箱即用地拥有现成的"ToString"方法，以可读的方式输出所有属性。

以下是使用记录定义相同值对象的方法：

```csharp
public record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
        {
            throw new ArgumentException("Invalid email address.", nameof(value));
        }

        Value = value;
    }
}
```

如果你不需要在 EmailAddress 和其他值对象内部进行验证，可以将其简化为：

```csharp
public record EmailAddress(string Value);
public record OrderNumber(string Value);
public record Address(string Street, string City, string Zip);
```

一行代码，简洁明了。

## 使用记录结构创建值对象

记录类型是值对象的绝佳选择，但它们是引用类型。如果你关心内存分配，可以使用 readonly record structs 来创建值对象。它们的行为与 record 相同，但它们是值类型，不会在堆上分配。

这是我创建值对象的个人选择。

以下是如何使用 record struct 定义值对象的方法：

```csharp
public readonly record struct EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains("@"))
        {
            throw new ArgumentException("Invalid email address.", nameof(value));
        }

        Value = value;
    }
}
```

或者以更简洁的形式：

```csharp
public readonly record struct EmailAddress(string Value);
public readonly record struct OrderNumber(string Value);
public readonly record struct Address(string Street, string City, string Zip);
```

以下是使用值对象后的 Customer 实体外观：

```csharp
public class Customer
{
    public Guid Id { get; private set; }
    public FirstName FirstName { get; private set; }
    public LastName LastName { get; private set; }
    public EmailAddress Email { get; private set; }
    public PhoneNumber PhoneNumber { get; private set; }
    public IReadOnlyList<Order> Orders => _orders.AsReadOnly();
    
    private readonly List<Order> _orders = [];
    
    private Customer() { }
    
    public static Customer Create(
        FirstName firstName,
        LastName lastName,
        EmailAddress email,
        PhoneNumber phoneNumber)
        {
            return new Customer
            {
                Id = Guid.NewGuid(),
                FirstName = firstName,
                LastName = lastName,
                Email = email,
                PhoneNumber = phoneNumber
            };
        }
        
    public void AddOrder(Order order)
    {
        _orders.Add(order);
    }
}
```

## 在 EF Core 中映射值对象

在实体模型中引入值对象后，你需要修改 EF Core 映射。你不能再像这样使用常规映射：

```csharp
builder.Property(x => x.FirstName).IsRequired();
builder.Property(x => x.LastName).IsRequired();
builder.Property(x => x.Email).IsRequired();
builder.Property(x => x.PhoneNumber).IsRequired();
```

你需要使用转换来告诉 EF Core 如何将值对象映射到数据库，以及如何将数据库值映射到值对象：

```csharp
builder.Property(x => x.Email)
    .HasConversion(
        email => email.Value,
        value => new EmailAddress(value)
    )
    .IsRequired();

builder.Property(x => x.PhoneNumber)
    .HasConversion(
        phoneNumber => phoneNumber.Value,
        value => new PhoneNumber(value)
    )
    .IsRequired();
```

它需要更多的代码，但值对象能给你带来很多优势。

## 值对象和请求/响应/DTO 模型

值对象是你的领域特定模型，外界不应该了解它们。此外，你的公共请求/响应/DTO 模型应该尽可能简单。

在请求/响应/DTO 模型中使用简单原始类型，并将它们映射到领域值对象，反之亦然，这是一种良好的实践。

例如，我正在我的"创建客户"用例中进行映射：

```csharp
var customer = Customer.Create(
    firstName: new FirstName(request.FirstName),
    lastName: new LastName(request.LastName),
    email: new EmailAddress(request.Email),
    phoneNumber: new PhoneNumber(request.PhoneNumber)
);

await customerRepository.AddAsync(customer, cancellationToken);
await unitOfWork.SaveChangesAsync(cancellationToken);
```

以及从值对象反向映射到 CustomerResponse ：

```csharp
internal static class MappingExtensions
{
    public static CustomerResponse MapToResponse(this Customer customer)
    {
        return new CustomerResponse(
            CustomerId: customer.Id,
            FirstName: customer.FirstName.Value,
            LastName: customer.LastName.Value,
            Email: customer.Email.Value,
            PhoneNumber: customer.PhoneNumber.Value
        );
    }
}
```

## 总结

如果你曾经历过代码中错误值进入数据库（或任何其他源）的 bug - 考虑在你的项目中使用值对象。C#记录和只读记录结构体为你提供了一种优雅、简单且快速的方式来实现值对象，而无需样板代码。

今天就到这里。期待与你再次相见。
