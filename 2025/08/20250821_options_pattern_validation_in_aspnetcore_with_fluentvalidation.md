# [使用 FluentValidation 在 ASP.NET Core 中进行选项模式验证](https://www.milanjovanovic.tech/blog/options-pattern-validation-in-aspnetcore-with-fluentvalidation)

如果你在 ASP.NET Core 中使用过**选项模式**，你可能熟悉使用数据注解的内置验证方式。虽然数据注解能够正常工作，但在复杂的验证场景中，它们可能会受到限制。

**选项模式**允许你在运行时使用类来获取强类型配置对象。

问题在于，在你尝试使用配置之前，你无法确定它是否有效。

那么为什么不在应用程序启动时验证它呢？

在本文中，我们将探讨如何将更强大的 FluentValidation 库与 ASP.NET Core 的**选项模式**集成，以构建一个在应用程序启动时执行的**健壮验证解决方案**。

## 为什么选择 FluentValidation 而非数据注解？

数据注解适用于简单验证，但 FluentValidation 提供了几个优势：

- 更具表现力和灵活性的验证规则
- 更好地支持复杂条件验证
- 更清晰的关注点分离（验证逻辑与模型分离）
- 更轻松的验证规则测试
- 更好的自定义验证逻辑支持
- 允许向验证器中注入依赖项

## 理解选项模式生命周期

在深入探讨验证之前，了解 ASP.NET Core 中选项的生命周期非常重要：

- 选项已在 DI 容器中注册
- 配置值会绑定到选项类
- 验证过程（如果已配置）
- 选项在通过 **IOptions\<T\>** 、 **IOptionsSnapshot\<T\>** 或 **IOptionsMonitor\<T\>** 请求时被解析

**ValidateOnStart()** 方法强制验证在应用程序启动时发生，而不是在首次解析选项时发生。

## 缺乏验证时的常见配置失败

如果没有验证，配置问题可能会以多种方式表现出来：

- **静默失败**：配置错误的选项可能会导致默认值被使用且不会发出警告
- **运行时异常**：当应用程序尝试使用无效值时，配置问题才可能显现
- **级联故障**：一个配置错误的组件可能导致依赖系统发生故障

通过在启动时进行验证，你可以创建一个快速反馈循环，从而防止这些问题。

## 奠定基础

首先，让我们将 FluentValidation 包添加到我们的项目中：

```bash
Install-Package FluentValidation # 基础包
Install-Package FluentValidation.DependencyInjectionExtensions # 与 DI 集成的扩展包
```

在我们的示例中，我们将使用一个 GitHubSettings 类，该类需要验证：

```csharp
public class GitHubSettings
{
    public const string ConfigurationSection = "GitHubSettings";

    public string BaseUrl { get;init; }
    public string AccessToken { get; init; }
    public string RepositoryName { get; init; }
}
```

## 创建一个 FluentValidation 验证器

接下来，我们将为我们的设置类创建一个验证器：

```csharp
public class GitHubSettingsValidator : AbstractValidator<GitHubSettings>
{
    public GitHubSettingsValidator()
    {
        RuleFor(x => x.BaseUrl).NotEmpty();

        RuleFor(x => x.BaseUrl)
            .Must(baseUrl => Uri.TryCreate(BaseUrl, UriKind.Absolute, out _))
            .When(x => !string.IsNullOrWhiteSpace(x.baseUrl))
            .WithMessage($"{nameof(GitHubSettings.BaseUrl)} must be a valid URL");

        RuleFor(x => x.AccessToken)
            .NotEmpty();

        RuleFor(x => x.RepositoryName)
            .NotEmpty();
    }
}
```

## 构建 FluentValidation 集成

要将 FluentValidation 与选项模式集成，我们需要创建一个自定义的 **IValidateOptions\<T\>** 实现：

```csharp
using FluentValidation;
using Microsoft.Extensions.Options;

public class FluentValidateOptions<TOptions>
    : IValidateOptions<TOptions>
    where TOptions : class
{
    private readonly IServiceProvider _serviceProvider;
    private readonly string? _name;

    public FluentValidateOptions(IServiceProvider serviceProvider, string? name)
    {
        _serviceProvider = serviceProvider;
        _name = name;
    }

    public ValidateOptionsResult Validate(string? name, TOptions options)
    {
        if (_name is not null && _name != name)
        {
            return ValidateOptionsResult.Skip;
        }

        ArgumentNullException.ThrowIfNull(options);

        using var scope = _serviceProvider.CreateScope();

        var validator = scope.ServiceProvider.GetRequiredService<IValidator<TOptions>>();

        var result = validator.Validate(options);
        if (result.IsValid)
        {
            return ValidateOptionsResult.Success;
        }

        var type = options.GetType().Name;
        var errors = new List<string>();

        foreach (var failure in result.Errors)
        {
            errors.Add($"Validation failed for {type}.{failure.PropertyName} " +
                       $"with the error: {failure.ErrorMessage}");
        }

        return ValidateOptionsResult.Fail(errors);
    }
}
```

关于此实现的几点重要说明：

1. 我们创建一个作用域服务提供程序来正确解析验证器（因为验证器通常注册为作用域服务）
2. 我们通过 _name 属性处理命名选项
3. 我们构建包含属性名称和错误消息的详细错误信息

## FluentValidation 集成的工作原理

在添加我们的自定义 FluentValidation 集成时，了解它如何连接到 ASP.NET Core 的选项系统会很有帮助：

1. **IValidateOptions\<T\>** 接口是 ASP.NET Core 为选项验证提供的钩子
2. 我们的 **FluentValidateOptions\<T\>** 类实现了这个接口以桥接到 FluentValidation
3. 当调用 **ValidateOnStart()** 时，ASP.NET Core 会解析所有 **IValidateOptions\<T\>** 实现并运行它们
4. 如果验证失败，会抛出一个 **OptionsValidationException** ，阻止应用程序启动

## 创建扩展方法以实现轻松集成

现在，让我们创建几个扩展方法，使我们的验证更易于使用：

```csharp
public static class OptionsBuilderExtensions
{
    public static OptionsBuilder<TOptions> ValidateFluentValidation<TOptions>(
        this OptionsBuilder<TOptions> builder)
        where TOptions : class
    {
        builder.Services.AddSingleton<IValidateOptions<TOptions>>(
            serviceProvider => new FluentValidateOptions<TOptions>(
                serviceProvider,
                builder.Name));

        return builder;
    }
}
```

这个扩展方法允许我们在配置选项时调用 **.ValidateFluentValidation()**，类似于内置的 **.ValidateDataAnnotations()** 方法。

为了更加方便，我们可以创建另一个扩展方法来简化整个配置过程：

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddOptionsWithFluentValidation<TOptions>(
        this IServiceCollection services,
        string configurationSection)
        where TOptions : class
    {
        services.AddOptions<TOptions>()
            .BindConfiguration(configurationSection)
            .ValidateFluentValidation() // 配置 FluentValidation 校验
            .ValidateOnStart(); // 在应用程序启动时验证选项

        return services;
    }
}
```

## 注册并使用验证

使用我们的 FluentValidation 集成有几种方法：

### 选项 1：标准注册与手动验证器注册

```csharp
// 注册校验规则
builder.Services.AddScoped<IValidator<GitHubSettings>, GitHubSettingsValidator>();

// 配置选项与验证
builder.Services.AddOptions<GitHubSettings>()
    .BindConfiguration(GitHubSettings.ConfigurationSection)
    .ValidateFluentValidation() // 配置 FluentValidation 校验
    .ValidateOnStart();
```

### 选项 2：使用便捷的扩展方法

```csharp
// 注册校验规则
builder.Services.AddScoped<IValidator<GitHubSettings>, GitHubSettingsValidator>();

// 使用便捷的扩展方法
builder.Services.AddOptionsWithFluentValidation<GitHubSettings>(GitHubSettings.ConfigurationSection);
```

### 选项 3：自动验证器注册

如果你有很多验证器并想一次性注册它们，可以使用 FluentValidation 的程序集扫描功能：

```csharp
// 注册所有验证器
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

// 使用便捷的扩展方法
builder.Services.AddOptionsWithFluentValidation<GitHubSettings>(GitHubSettings.ConfigurationSection);
```

## 运行时会发生什么？

使用 **.ValidateOnStart()** 时，如果任何验证规则失败，应用程序将在启动期间抛出异常。例如，如果你的 appsettings.json 缺少必需的 AccessToken ，你将会看到类似以下内容：

```bash
Microsoft.Extensions.Options.OptionsValidationException:
    Validation failed for GitHubSettings.AccessToken with the error: 'Access Token' must not be empty.
```

这可以防止应用程序在配置无效的情况下启动，确保问题能够尽早被发现。

## 使用不同的配置源

ASP.NET Core 的配置系统支持多个源。当使用 FluentValidation 的选项模式时，请记住无论配置源是什么，验证都会正常工作：

- 环境变量
- Azure键保管库
- 用户机密
- JSON 文件
- 内存配置

这对于容器化应用特别有用，因为这类应用的配置来自环境变量或挂载的机密信息。

## 测试你的验证器

使用 FluentValidation 的一个好处是验证器易于测试：

```csharp
// 使用 FluentValidation.TestHelper 的辅助方法
[Fact]
public void GitHubSettings_WithMissingAccessToken_ShouldHaveValidationError()
{
    // 分配
    var validator = new GitHubSettingsValidator();
    var settings = new GitHubSettings { RepositoryName = "test-repo" };

    // 执行
    TestValidationResult<GitHubSettings>? result = await validator.TestValidate(settings);

    // 断言
    result.ShouldNotHaveAnyValidationErrors();
}
```

## 总结

通过将 FluentValidation 与选项模式和 **ValidateOnStart()** 相结合，我们创建了一个强大的验证系统，确保应用程序在启动时拥有正确的配置。

这种方法：

1. 提供比数据注解更具表现力的验证规则
2. 将验证逻辑与配置模型分离
3. 在应用程序启动时捕获配置错误
4. 支持复杂的验证场景
5. 易于测试

这种模式在微服务架构或容器化应用中特别有价值，因为配置错误应该被立即检测出来，而不是在运行时才发现。

记得适当注册你的验证器，并使用 **.ValidateOnStart()** 来确保验证在应用程序启动时发生。

今天就到这里。期待与你再次相见。
