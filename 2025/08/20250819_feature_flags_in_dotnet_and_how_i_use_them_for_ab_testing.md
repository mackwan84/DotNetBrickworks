# [使用 .NET 中的功能标志进行 A/B 测试](https://www.milanjovanovic.tech/blog/feature-flags-in-dotnet-and-how-i-use-them-for-ab-testing)

在应用程序中无需重新部署代码即可有条件地开启或关闭功能的能力是一种强大的工具。

它让你能够快速迭代新功能，并频繁地将你的更改集成到主分支中。

你可以使用**功能标志**来实现这一点。

**功能标志**是一种软件开发技术，允许你将应用程序功能包装在条件语句中。然后你可以在运行时切换功能的开启或关闭，以控制哪些功能被启用。

今天我们将讨论：

- .NET 中的功能标志基础
- 功能筛选和分阶段发布
- 基于主干开发
- A/B 测试

闲话少说，立即开始吧！

## .NET 中的功能标志

**功能标志**为 .NET 和 ASP.NET Core 应用程序提供了一种动态开启或关闭功能的方式。

要开始使用，你需要在项目中安装 **Microsoft.FeatureManagement** 库：

```bash
Install-Package Microsoft.FeatureManagement
```

该库将允许你基于功能来开发和展示应用程序功能。当你对何时启用新功能以及启用条件有特殊要求时，它非常有用。

下一步是通过调用 **AddFeatureManagement** 来注册所需的依赖注入服务：

```csharp
builder.Services.AddFeatureManagement();
```

现在你已经准备好创建你的第一个功能标志。功能标志构建在.NET 配置系统之上。任何.NET 配置提供程序都可以作为暴露功能标志的基础。

让我们在 **appsettings.json** 文件中创建一个名为 **ClipArticleContent** 的功能标志：

```json
{
  "FeatureManagement": {
    "ClipArticleContent": true
  }
}
```

按照惯例，功能标志必须在 **FeatureManagement** 配置节中定义。但是，你可以通过在调用 **AddFeatureManagement** 时提供不同的配置节来更改这一点。

Microsoft 建议使用枚举来暴露功能标志，然后使用 **nameof** 运算符来使用它们。例如，你会编写 **nameof(FeatureFlags.ClipArticleContent)** 。

然而，我更喜欢在静态类中将功能标志定义为常量，因为这简化了使用方式。

```csharp
public static class FeatureFlags
// 使用枚举定义
public enum FeatureFlags
{
    ClipArticleContent = 1
}

// 使用常量定义
public static class FeatureFlags
{
    public const string ClipArticleContent = "ClipArticleContent";
}
```

要检查功能标志状态，你可以使用 **IFeatureManager** 服务。在这个示例中，如果 **ClipArticleContent** 功能标志被打开，我们将只返回文章内容的前三十个字符。

```csharp
app.MapGet("articles/{id}", async (
    Guid id,
    IGetArticle query,
    IFeatureManager featureManager) =>
{
    var article = query.Execute(id);

    if (await featureManager.IsEnabledAsync(FeatureFlags.ClipArticleContent))
    {
        article.Content = article.Content.Substring(0, 50);
    }

    return Results.Ok(article);
});
```

你还可以使用 **FeatureGate** 属性在控制器或端点级别应用功能标志：

```csharp
[FeatureGate(FeatureFlags.EnableArticlesApi)]
public class ArticlesController : Controller
{
   // ...
}
```

这涵盖了使用功能标志的基础知识，现在，让我们来探讨更高级的主题。

## 功能过滤器与分阶段发布

我在上一节向你展示的功能标志就像一个简单的开关。虽然实用，但你可能希望从功能标志中获得更多的灵活性。

**Microsoft.FeatureManagement** 包附带了一些内置的功能过滤器，允许你创建用于启用功能标志的动态规则。

可用的功能筛选器有 **Microsoft.Percentage** 、 **Microsoft.TimeWindow** 和 **Microsoft.Targeting** 。

以下是一个定义使用百分比过滤器的 **ShowArticlePreview** 功能标志的示例：

```json
{
  "FeatureFlags": {
    "ClipArticleContent": false,
    "ShowArticlePreview": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 50
          }
        }
      ]
    }
  }
}
```

这意味着功能标志将随机开启 50% 的时间。缺点是同一用户在后续请求中可能会看到不同的行为。更现实的场景是在用户会话期间缓存功能标志的状态。

要使用 **PercentageFilter** ，你需要通过调用 **AddFeatureFilter** 来启用它：

```csharp
builder.Services.AddFeatureManagement().AddFeatureFilter<PercentageFilter>();
```

另一个有趣的功能过滤器是 **TargetingFilter** ，它允许你针对特定用户。定向用于分阶段推出，你希望逐步向用户引入新功能。你首先为小部分用户启用该功能，然后在监控系统响应的同时慢慢增加推出百分比。

## 基于主干开发和功能标志

主干开发是一种 Git 分支策略，其中所有开发人员在短期分支中工作或直接在主干（主代码库）中工作。"主干"是你存储库的主分支。如果你使用 Git，它将是 main 或 master 分支。主干开发避免了由长期分支引起的"合并地狱"问题。

那么，功能标志在基于主干的开发中如何发挥作用呢？

确保主干始终可发布的唯一方法是将未完成的功能隐藏在功能标志后面。在开发功能时，你继续向主干推送更改，同时功能标志在主分支上保持关闭状态。当功能完成后，你打开功能标志并将其发布到生产环境。

![基于主干开发](https://www.milanjovanovic.tech/blogs/mnw_055/trunk_based_development.png?imwidth=1920)

## 我如何在我的网站上使用特性标志进行 A/B 测试

A/B 测试（分割测试）是一种实验，其中页面的两个或多个变体（或功能）会随机展示给用户。系统会在后台进行统计分析，以确定哪种变体对于给定的转化目标表现更好。

这是我在我的网站上执行的一个 A/B 测试示例：

![A / B 测试](https://www.milanjovanovic.tech/blogs/mnw_055/split_test.png?imwidth=1920)

假设是移除图片并专注于产品优势会让更多人想要订阅。我通过转化率来衡量这一点，转化率是指访问页面的人数除以订阅人数的比例。

我正在使用一个名为 Posthog 的平台来运行实验，该平台会自动计算结果。

你可以看到测试变体的转化率显著更高，因此它成为了 A/B 测试的获胜者。

![测试结果](https://www.milanjovanovic.tech/blogs/mnw_055/experiment_results.png?imwidth=1920)

## 结论

无需部署代码就能动态开启或关闭功能的能力简直就像超能力。而功能标志只需很少的工作就能让你拥有这种能力。

你可以通过安装 **Microsoft.FeatureManagement** 库在.NET 中使用功能标志。功能标志构建于 .NET 配置系统之上，你可以使用 **IFeatureManager** 服务来检查功能标志状态。

功能标志的另一个用例是 A/B 测试。我在我的网站上每周运行实验，测试能够提高转化率的变更。功能标志帮助我决定向用户展示哪个版本的网站。然后，我可以根据用户的行为来衡量结果。

今天就到这里。期待与你再次相见。
