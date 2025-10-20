# [如何使用 .NET 9 为整洁架构编写架构测试](https://medium.com/@mariammaurice/%EF%B8%8F-how-to-write-architecture-tests-for-clean-architecture-with-net-9-6da97ae8ca64)

## 引言：为什么架构测试很重要

软件起初是干净、整洁、有结构的。但随着时间的推移，随着众多贡献者的加入、需求的变更、截止日期的压力和技术债务的累积，它往往会偏离初衷。

然而问题逐渐浮现：

- 内层（例如领域层）意外地依赖于外层（例如基础设施层或表示层）。
- 仓储或 ORM/文件存储代码泄露到领域层。
- 业务逻辑与控制器或 UI 混合在一起。
- 命名约定被破坏，依赖关系变得混乱。

架构测试是自动化检查，它将你的架构规则（分层、依赖方向、命名、可见性）编码化并持续执行。因此，当有人意外引入违规行为时，测试套件会失败，问题会被及早发现，而不是在多次合并后或生产环境中才发现。

## 整洁架构与 .NET 9 — 层与依赖规则

在编写测试之前，你需要对架构有清晰的定义。在大多数整洁架构/洋葱架构/六边形架构风格中，你的项目/层看起来如下：

- 领域（核心业务逻辑、实体、值对象、领域事件）
- 应用程序（用例、接口/端口）
- 基础设施（EF Core/持久化、外部服务、文件/队列等）
- 展示层/API/UI（控制器、端点、GraphQL 等）

依赖规则（典型）：

- 领域层 → 不依赖其他层（不引用应用层、基础设施层或表现层）
- 应用层 → 依赖领域层，但不依赖基础设施层或表现层（如果遵循严格的选择）
- 基础设施层 → 依赖应用层和领域层，但反之则不然
- 表现层 → 依赖应用层 → 依赖领域层
- 命名约定（例如，仓储类以"Repository"结尾，领域事件以"Event"或"DomainEvent"结尾等）
- 领域中的类通常应该是密封的/不可变的（取决于你的风格）

清晰地记录你的规则。这将成为你的架构测试的基准

## 整洁架构中的测试类型

让我们对测试进行分类，看看架构测试适合在哪里：

![测试类型](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gqfo3p0qhadco5O_Jqp5Jw.png)

架构测试可以减少偏差，帮助在系统增长时保持其正确形态，特别是在由许多开发者共同开发的情况下。

## 工具与库

以下是一些有助于在 .NET 中编写架构测试的工具：

- NetArchTest.Rules — 非常流行且富有表现力，允许你断言依赖关系、类型等。
- ArchUnitNET — 灵感来自 Java 的 ArchUnit，也提供类型/依赖关系检查。（虽然不总是被广泛使用，但非常有用）
- 反射/自定义代码 — 如果现有工具无法满足你的需求，可用于定制规则。

使用 NetArchTest 时，典型模式是：加载程序集，使用过滤器（命名空间、类与接口、名称模式）选择类型，然后断言特定的依赖关系或命名规则。EzzyLearning.net+1

## 设计架构测试 — 需要强制执行的内容

以下是一些常见的规则/约束，可以将其编码为架构测试：

1. 分层约束
    - 领域层不能引用应用层、基础设施层、表现层
    - 应用程序不得引用基础设施层（根据具体风格而定）
    - 表示层只能依赖于应用程序层（可能还包括一些领域抽象）
2. 依赖方向
    - 如果应用程序层仅通过接口依赖于基础设施层，请确保实现细节位于基础设施层中
3. 命名约定
    - 特定目录/命名空间中的类必须遵循命名规范——例如，仓储类以 Repository 结尾
    - 领域事件命名为 SomethingEvent 或 SomethingDomainEvent
4. 可见性/可变性约束
    - 领域类应为密封类（如果你的编码风格要求不可变性）
    - 实体应具有私有无参构造函数（用于 ORM）
    - 避免在实体/值对象中使用公共设置器，除非必要
5. 强制执行接口/抽象
    - 基础设施实现必须实现应用程序/域中定义的接口
    - 应用程序/域中不得直接使用具体的基础设施类
6. 命名空间/项目中的层边界
    - 确保命名空间/项目反映各层且依赖关系匹配
7. 避免在领域层中使用特定类型
    - 例如，在领域层中不要引用 EF Core（DbContext 或 EF 特性）
8. 确保依赖注入的使用
    - 控制器/表示层应该通过接口获取依赖，而不是具体类型（可以进行表面测试）

你应该选择你关心且团队同意的子集。

## 示例项目设置（ .NET 9 ）

为了说明，我们假设一个示例项目结构：

```
/src
   /Domain         → Domain project (Class Library)  
       Entities, ValueObjects, DomainEvents, Interfaces  
   /Application    → Use Cases, DTOs, Interfaces  
   /Infrastructure → EF Core, Repositories, external services  
   /WebApi         → ASP.NET Core Web API, Controllers  
/tests
   /ArchitectureTests → separate test project for architecture tests  
   /DomainTests       → unit tests for domain logic  
   /ApplicationTests  → unit tests for use case logic  
   /IntegrationTests  → tests involving database, etc.
```

依赖项：

- Domain 项目不依赖于 Application、Infrastructure、WebApi
- Application 引用 Domain
- Infrastructure 引用 Application 和 Domain
- WebApi 引用 Application

此外，可能还包括一个共享内核或通用抽象项目，视情况而定。

## 编写架构测试：真实代码示例

以下是在 .NET 9 中使用 NetArchTest.Rules 的具体示例。

### 设置

在你的测试解决方案中，添加一个名为 MyApp.ArchitectureTests （或类似名称）的项目。添加对测试项目的引用，并添加 NuGet 包：

```bash
dotnet add MyApp.ArchitectureTests package NetArchTest.Rules
dotnet add MyApp.ArchitectureTests package FluentAssertions
```

你需要引用你想要测试的程序集，例如：

```xml
<ItemGroup>
  <ProjectReference Include="..\src\Domain\Domain.csproj" />
  <ProjectReference Include="..\src\Application\Application.csproj" />
  <ProjectReference Include="..\src\Infrastructure\Infrastructure.csproj" />
  <ProjectReference Include="..\src\WebApi\WebApi.csproj" />
</ItemGroup>
```

### 示例：领域层不依赖于其他层

```csharp
using NetArchTest.Rules;
using System.Reflection;
using Xunit;
using FluentAssertions;
public class LayeringTests
{
    private readonly Assembly _domainAssembly = typeof(SomeDomainEntity).Assembly;
    private readonly Assembly _applicationAssembly = typeof(SomeApplicationService).Assembly;
    private readonly Assembly _infrastructureAssembly = typeof(SomeRepositoryImplementation).Assembly;
    [Fact]
    public void Domain_Layer_Should_Not_Have_Dependency_On_Application_Or_Infrastructure()
    {
        // Arrange
        var result = Types.InAssembly(_domainAssembly)
            .ShouldNot()
            .HaveDependencyOn(_applicationAssembly.GetName().Name)
            .AndNot()
            .HaveDependencyOn(_infrastructureAssembly.GetName().Name)
            .GetResult();
        // Assert
        result.IsSuccessful.Should().BeTrue("Domain layer should not depend on Application or Infrastructure layers");
    }
}
```

### 示例：应用层不应直接依赖于基础设施层

```csharp
[Fact]
public void Application_Layer_Should_Not_Have_Dependency_On_Infrastructure()
{
    var result = Types.InAssembly(_applicationAssembly)
        .ShouldNot()
        .HaveDependencyOn(_infrastructureAssembly.GetName().Name)
        .GetResult();
    result.IsSuccessful.Should().BeTrue("Application layer should should not depend on Infrastructure directly");
}
```

### 示例：仓储类命名约定

```csharp
[Fact]
public void Repository_Implementations_Should_End_With_Repository()
{
    var infraTypes = Types
        .InAssembly(_infrastructureAssembly)
        .That()
        .ResideInNamespace("MyApp.Infrastructure.Repositories")
        .And()
        .AreClasses()
        .GetTypes();
    foreach(var t in infraTypes)
    {
        t.Name.Should().EndWith("Repository", $"Class {t.Name} should be named with *Repository suffix");
    }
}
```

或者使用 NetArchTest：

```csharp
[Fact]
public void RepositoryClasses_Should_Have_NameEndingWith_Repository()
{
    var result = Types.InAssembly(_infrastructureAssembly)
        .That()
        .ResideInNamespace("MyApp.Infrastructure.Repositories")
        .And()
        .AreClasses()
        .Should()
        .HaveNameEndingWith("Repository")
        .GetResult();
    result.IsSuccessful.Should().BeTrue();
}
```

### 示例：领域事件应该是密封的/不可变的

```csharp
[Fact]
public void DomainEvents_Should_Be_Sealed()
{
    var domainEvents = Types.InAssembly(_domainAssembly)
        .That()
        .HaveNameEndingWith("Event")
        .And()
        .AreClasses()
        .GetResult();
    foreach(var t in domainEvents.Types)
    {
        t.IsSealed.Should().BeTrue($"{t.FullName} should be sealed to enforce immutability / design style");
    }
}
```

### 示例：领域实体中不应有公共设置器

```csharp
[Fact]
public void Domain_Entities_Should_Not_Have_Public_Setters()
{
    var entityTypes = Types.InAssembly(_domainAssembly)
        .That()
        .ResideInNamespace("MyApp.Domain.Entities")
        .GetTypes();
    foreach(var t in entityTypes)
    {
        var props = t.GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.SetMethod != null && p.SetMethod.IsPublic);
        props.Should().BeEmpty($"{t.Name} should not expose public setters on its properties");
    }
}
```

### 示例：控制器应使用应用层，而非直接使用领域层

```csharp
[Fact]
public void Controllers_Should_Not_Depend_On_Domain_Entities_Directly()
{
    var webApiAssembly = typeof(SomeController).Assembly;
    var domainAssemblyName = _domainAssembly.GetName().Name;
    var result = Types.InAssembly(webApiAssembly)
        .That()
        .ResideInNamespace("MyApp.WebApi.Controllers")
        .ShouldNot()
        .HaveDependencyOn(domainAssemblyName)
        .GetResult();
    result.IsSuccessful.Should().BeTrue("Controllers should go through Application layer (DTOs, interfaces), not depend directly on domain types");
}
```

## 在 CI/CD 中运行架构测试

要使架构测试有效：

- 将它们添加到你的构建流水线中（GitHub Actions、Azure DevOps 等）。如果测试失败，则阻止合并。
- 让它们保持快速！因为它们是结构性的测试，通常运行很快（基于反射，不涉及数据库）。
- 保持命名和依赖规则稳定；更改这些规则应该是一个经过深思熟虑的决定。当你更改架构决策时，相应地更新测试。
- 将架构测试覆盖率纳入你的代码质量指标中。

GitHub Actions 示例片段：

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup .NET 9
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '9.x'
      - name: Restore
        run: dotnet restore
      - name: Build
        run: dotnet build --no-restore
      - name: Run unit tests
        run: dotnet test --no-restore --verbosity normal
      - name: Run architecture tests
        run: dotnet test --filter Category=Architecture --no-restore --verbosity normal
```

你可以为架构测试项目或测试添加自定义标签（例如 [Trait("Category","Architecture")] ），以便单独运行或筛选它们。

## 权衡、陷阱与最佳实践

编写架构测试虽然能带来价值，但也有一些需要注意的地方。

### 权衡/成本

- 前期工作：你需要定义规则、编写测试，并在架构演变时维护这些测试。
- 误报/脆弱规则：过于严格的命名或可见性规则可能会激怒开发人员或阻止有效的设计变更。
- 维护开销：当风格或决策发生变化时，许多测试可能需要更新。

### 常见陷阱

- 过早编写过多规则；某些规则属于过早优化。
- 过度限制系统的次要部分，导致开发摩擦。
- 未保持测试的快速性；如果架构测试运行缓慢或不稳定，人们会禁用或忽略它们。
- 将行为测试与架构测试混合（保持分离）。

### 最佳实践

- 从一小套核心架构规则开始（层依赖关系，禁止领域层→外部）。
- 使用像 NetArchTest 这样的工具来清晰地表达规则。
- 为架构测试打上标签，以便清晰区分。
- 经常运行它们（持续集成）。
- 记录你的架构决策；保持规则的版本控制。
- 当架构发生合理变更时，更新测试并传达变更信息。
- 让团队就约定（命名、分层）达成一致，这样测试就能反映共同的理解，而不是武断的规则。

## 结论

架构测试是维护整洁架构代码库完整性的强大工具。它们将设计约束编码化，防止架构侵蚀，并帮助新团队成员无需阅读所有文档就能遵守规则。

以下是关键要点：

- 整洁架构是关于分层和依赖方向；架构测试则强制执行这一点。
- 在 .NET 9 中使用 NetArchTest.Rules 等工具来编写基于反射的程序集测试。
- 选择对你的项目重要的规则（分层、命名、可见性、依赖关系）。
- 将架构测试集成到 CI/CD 中。
- 要预见到架构会不断演进；测试和规则不仅需要代码维护，它们本身也需要维护。

如果你尽早构建架构测试（或很快集成它们），随着团队规模的扩大，它们将在减少技术债务和保持结构清洁方面带来巨大回报。

今天就到这里。期待与你再次相见。
