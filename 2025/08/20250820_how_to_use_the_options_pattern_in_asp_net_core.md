# [如何在 ASP.NET Core 中使用选项模式](https://www.milanjovanovic.tech/blog/how-to-use-the-options-pattern-in-asp-net-core-7)

今天，我想向你展示如何在 ASP.NET Core 中使用强大的选项模式。

**选项模式**使用类在应用程序运行时提供**强类型设置**。

**选项**实例的值可以来自多个来源。典型用例是从应用程序配置中提供设置。

在 ASP.NET Core 中，你可以通过几种不同的方式来配置选项模式。我想讨论其中的一些方法及其潜在优势。

闲话不多说，让我们开始吧！

## 创建选项类

首先，我想通过创建**选项**类并解释我们想要绑定到它的设置来做好准备。

我们想要为应用程序配置 JWT 身份验证，因此我们决定创建 **JwtOptions** 类来保存该配置：

```csharp
public class JwtOptions
{
    public string Issuer { get; set; }
    public string Audience { get; set; }
    public string Secret { get; set; }
}
```

让我们想象一下，在我们的 appsettings.json 文件中，我们有以下配置值：

```json
{
  "Jwt": {
    "Issuer": "your-issuer",
    "Audience": "your-audience",
    "Secret": "your-secret"
  }
}
```

好的，看起来不错。现在我想向你展示几种将 JSON 中的值绑定到我们的 **JwtOptions** 类的方法。

## 使用 IConfiguration 设置选项模式

最直接的方法是使用我们在注册服务时可以访问的 IConfiguration 实例。

我们需要调用 **IServiceCollection.Configure\<TOptions\>** 方法，并将 **JwtOptions** 指定为泛型参数：

```csharp
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection("Jwt"));
```

这还能更简单吗？

唯一的缺点是我们仅限于通过应用程序配置提供的配置值。

这可以扩展为同时包含环境变量和用户机密。

## 使用 IConfigureOptions 设置选项模式

如果你想要一个更稳健的方法，我已经为你准备好了。我们将使用 **IConfigureOptions** 接口来定义一个类，以配置我们的**强类型选项**。

在这种情况下，我们需要遵循两个步骤：

- 创建 **IConfigureOptions** 实现
- 使用我们的 **IConfigureOptions** 实现作为泛型参数调用 **IServiceCollection.ConfigureOptions\<TOptions\>**

首先，我们将创建 **JwtOptionsSetup** 类：

```csharp
public class JwtOptionsSetup : IConfigureOptions<JwtOptions>
{
    private const string SectionName = "Jwt";
    private readonly IConfiguration _configuration;

    public JwtOptionsSetup(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void Configure(JwtOptions options)
    {
        _configuration
            .GetSection(SectionName)
            .Bind(options);
    }
}
```

我们写了更多的代码，来实现相同的事情。这样做值得吗？

也许，如果你考虑到我们现在可以在 **JwtOptionsSetup** 类中访问依赖注入。这意味着我们可以解析其他服务，用来获取配置值。

我们还需要告诉应用程序使用 **JwtOptionsSetup** 类：

```csharp
builder.Services.ConfigureOptions<JwtOptionsSetup>();
```

当我们尝试在某处注入 **JwtOptions** 时， **JwtOptionsSetup.Configure** 方法将首先被调用以计算正确的值。

## 使用 IOptions 注入选项

我们已经看到了几个如何使用 **JwtOptions** 类配置选项模式的示例。

但我们究竟该如何使用**选项模式**呢？

很简单，你只需要从构造函数中注入 **IOptions\<JwtOptions\>** 。

为简洁起见，我这里只展示 **JwtProvider** 构造函数。

```csharp
public class JwtProvider
{
    private readonly JwtOptions _options;

    public JwtProvider(IOptions<JwtOptions> options)
    {
        _options = options.Value;
    }
}
```

实际的 **JwtOptions** 实例在 **IOptions\<JwtOptions\>.Value** 属性上可用。

我们在这里注入的 **IOptions** 实例在依赖注入中被配置为单例。意识到这一点非常重要。

## IOptionsSnapshot 和 IOptionsMonitor 呢？

如果你希望在每次注入选项类时都使用最新的配置值，那么注入 **IOptions** 将不起作用。

但是，你可以使用 **IOptionsSnapshot** 接口来替代：

- 它提供最新的配置快照（每个请求缓存）
- 它被注册为 **Scoped** 服务
- 它会检测应用程序启动后的配置更改

你也可以使用 **IOptionsMonitor** ，它可以在任何时候检索当前的选项值，并且它是一个 Singleton 服务。

## 总结

**选项模式**为我们提供了一种在应用程序中使用强类型配置类的方法。

我们可以通过 **IConfiguration** 以简单方式配置选项类，如果需要更强大的功能，也可以创建一个 **IConfigureOptions** 实现。

当涉及到使用**选项模式**时，我们有三种方法：

- IOptions
- IOptionsSnapshot
- IOptionsMonitor

在应用程序中决定使用哪种方法取决于你想要的行为类型。如果你不需要在应用程序启动后支持刷新配置值， **IOptions** 是一个完美的解决方案。

今天就到这里。期待与你再次相见。
