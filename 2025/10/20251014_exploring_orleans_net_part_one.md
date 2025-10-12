# [探索 Orleans.NET - 第一部分](https://medium.com/@ZaradarTR/exploring-orleans-net-55ddaaef3fc5)

在当今的软件开发领域，对可扩展且具有韧性的分布式应用程序的需求急剧增长。现代应用程序必须处理海量数据、保持高可用性，并能够无缝扩展以满足用户需求，同时还要确保强大的性能和可靠性。这种日益增长的复杂性需要新的框架来简化开发过程，同时满足这些严格的要求。Orleans.NET 应运而生，这是一个由微软设计的框架，旨在提高开发人员生产力并简化高性能系统的创建。它通过提供一组强大的抽象、保证和系统服务，为构建分布式系统提供了独特的方法，从根本上改变了开发人员应对分布式应用程序开发挑战的方式。

该框架的设计直观且易于上手，使其成为专家和新手程序员的宝贵工具。通过抽象分布式系统的复杂性，Orleans.NET 让开发者能够专注于业务逻辑，而无需处理并发、状态管理和容错等繁琐细节。这种抽象显著减少了开发时间和精力，从而能够快速迭代并部署稳健的应用程序。

微软自身在其产品中也广泛使用 Orleans.NET，这证明了该框架的可靠性和可扩展性。值得注意的是，Orleans.NET 为 Microsoft Azure 和 Xbox Live 中的多项关键服务提供支持，以高可用性和高性能处理每秒数百万的请求。这种内部使用，通常被称为“吃自己的狗粮”，确保了 Orleans.NET 在真实世界条件下得到持续的测试和改进，并得益于微软丰富的运营经验和严格的质量标准。

![Orleans.NET](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*WqgOofplkvREg1yf.png)

## 主要特性

Orleans.NET 允许开发者构建超越单一进程和硬件边界的应用程序，通过使用对等通信实现高可用性和弹性。其实现方式是利用熟悉的 C# 概念和模式来简化开发者从单服务器环境到多服务器环境的过渡，并聚焦于于提升其适用性和易用性的关键功能：

1. 可扩展性：设计用于从单个本地服务器扩展到云中数百或数千个分布式应用程序。该框架支持弹性扩展，这意味着新主机可以加入集群以承担额外负载，而现有主机也可以在不影响应用程序整体功能的情况下离开。
2. 容错性：集群会自动检测主机故障，并在剩余的主机之间重新分配工作负载，从而确保持续可用性和可靠性。
3. 跨平台支持：可在任何支持 .NET 的环境中运行，包括 Linux、Windows 和 macOS。它支持多种托管环境，如 Kubernetes、虚拟机，以及平台即服务 (PaaS) 产品，如 Azure App Service 和 Azure Container Apps。
4. 简化的分布式应用程序开发：Orleans .NET 的主要目标之一是简化分布式系统的复杂性。它提供了一套通用的模式和 API，使开发者能够构建具有弹性和可扩展性的云原生服务，而无需管理分布式计算的复杂细节。
5. Actor 模型：基于 20 世纪 70 年代的“actor 模型”这一编程范式构建。每个 actor（称为“Grain”）是一个轻量级、并发、不可变的对象，负责处理状态和行为，并通过异步消息进行通信。该模型通过确保 actor 即使在服务器故障期间也能永久存在，从而增强了可扩展性和容错能力。

![Actor 模型](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*_C3WOAMbwWN6Fx82lpEhZw.png)

## 什么是 Grains？

在 Orleans .NET 中，其架构的基石在于 Grain，这是一种复杂而强大的抽象，类似于虚拟 actor。Grain 体现了用户定义的身份、行为和状态，构成了构建分布式系统的基础。每个 Grain 始终准备好被调用，无论是被其他 Grain 还是外部客户端召唤，都遵循特定接口，如 IGrainWithGuidKey、IGrainWithIntegerKey 或 IGrainWithStringKey，这些接口明确了 Grain 被识别和访问的机制。

在 Grains 的领域内，具备承载易失性和持久性状态数据的能力，这些数据可以存储在多样化的存储系统中。这种灵活的设计使 Grains 能够自动分割应用程序状态，从而增强可扩展性并简化故障恢复协议。在活动状态下，Grain 数据驻留在内存中，从而促进快速处理、最小化延迟，并减轻底层数据存储的负担。Grain 生命周期的这种编排由 Orleans 运行时巧妙地管理，根据操作需求在激活和停用之间无缝切换，从而使开发人员可以奢侈地在假设所有 Grains 永久存在于内存中的前提下编写代码。

![Gains 生命周期](https://miro.medium.com/v2/resize:fit:646/format:webp/1*6eyU5WAkSxzP0W8S4kay6g.png)

与其理论基础相辅相成，Orleans 拥有实际应用，以 Mardi Gras .NET 参考应用程序为例，其中 Person Grain 是 Orleans 实用性、易用性和适用性的一个实际范例，如下面的代码片段所示：

```csharp
using Domain.Interfaces;

namespace Domain.Grains;

public class Person(ILogger<Person> logger) : IPerson
{
    private readonly ILogger _logger = logger;

    public string Name { get; private set; } = string.Empty;

    public int Energy { get; private set; } = 100;

    ValueTask<string> IPerson.Do(ActionType action)
    {
        switch (action)
        {
            case ActionType.Eat:
                Eat();
                break;
            case ActionType.Drink:
                Drink();
                break;
            case ActionType.Party:
                Party();
                break;
            case ActionType.Sleep:
                Sleep();
                break;
            default:
                throw new ArgumentException("Invalid action type.");
        }

        return ValueTask.FromResult($"{Name} has {Energy} energy left after {action}.");
    }

    ValueTask<string> IPerson.GetName()
    {
        return ValueTask.FromResult(Name);
    }

    Task IPerson.SetName(string value)
    {
        Name = value;

        return Task.CompletedTask;
    }

    void Eat()
    {
        _logger.LogInformation($"{Name} is eating.");

        Energy += 10;
        
        if (Energy > 100) Energy = 100;
    }

    void Drink()
    {
        if(Energy - 5 <= 0) throw new InvalidOperationException($"{Name} does not have enough energy to drink.");
        
        _logger.LogInformation($"{Name} is drinking.");
        
        Energy -= 5;
    }

    void Party()
    {
        if(Energy - 20 <= 0) throw new InvalidOperationException($"{Name} does not have enough energy to party.");

        _logger.LogInformation($"{Name} is partying.");
        
        Energy -= 20;
    }

    void Sleep()
    {
        _logger.LogInformation($"{Name} is sleeping.");
        
        Energy += 30;

        if (Energy > 100) Energy = 100;
    }

    public override string ToString()
    {
        return $"{Name} (Energy: {Energy})";
    }
}
```

## 什么是 Silo？

Silo 是 Orleans 框架中 Grain 的基本容器。作为 Grain 的执行环境，Silo  负责其激活、停用和通信。在集群中，Silo  协同工作，平衡负载并确保资源的高效利用。它们为托管的 Grain 提供强大的运行时环境，提供定时器、持久化、事务等基本服务。

Grain 作为计算的基本单元，依赖 Silo  进行执行和管理。Silo 抽象了分布式系统的复杂性，使 Grain 能够无缝运行，如同在单个进程中一样。它们通过与集群中的其他 Silo  协调，确保 Grain 的可靠性和可扩展性。通过 Silo ，Grain 可以进行异步通信，从而开发出响应迅速且可扩展的应用程序。

![Silo 架构](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*_Qf7D4WF4xCTv1J71ML6zA.png)

Silo 在容错方面也发挥着关键作用，与其他 Silo 协作检测故障并优雅地恢复。它们为 Grain 提供了弹性环境，确保即使在节点故障或网络分区的情况下也能持续运行。借助 Silo ，开发者可以专注于应用逻辑，而无需担心底层基础设施问题。

简而言之， Silo 是 Orleans 框架的支柱，为 Grain 提供执行环境和运行时服务。它们通过抽象化分布式系统的复杂性，支持开发可扩展、容错的分布式应用程序。通过 Silo ， Grain 可以在集群内无缝通信并高效运行，确保分布式应用程序的可靠性和响应性。

在 Orleans 中设置包含独立进程的本地集群十分简单，开发人员可以轻松地创建本地集群环境，用于测试和开发。使用下面的代码片段，开发人员可以初始化一个主机生成器，配置使用 localhost 聚类的 Orleans 独立进程，并指定日志设置。此设置使 Grains 能够在本地集群中运行，让开发人员可以专注于编写应用程序逻辑，而无需担心分布式系统的复杂性：

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

IHostBuilder builder = Host.CreateDefaultBuilder(args)
    .UseOrleans(silo =>
    {
        silo.UseLocalhostClustering()
            .ConfigureLogging(logging => logging.AddConsole());
    })
    .UseConsoleLifetime();

using IHost host = builder.Build();

await host.RunAsync();
```

## 总结

Orleans.NET 确实是一个构建可扩展且具有弹性的分布式应用程序的引人注目的框架。其以 Grain 和独立进程概念为核心的架构，为开发人员提供了强大的抽象和工具，以简化开发过程并创建健壮的系统。

Grains 作为虚拟 Actor，封装了行为和状态，使开发人员可以专注于定义业务逻辑，而不是管理分布式计算的复杂性。借助 Orleans，Grains 可以被轻松地激活、停用和通信，为开发人员提供了无缝的体验。

另一方面，Silo 在 Orleans 框架中充当 Grain 的执行环境。它们抽象了分布式系统的复杂性，提供必要的运行时服务，并确保 Grain 的可靠性和可扩展性。Silo 在容错方面发挥着关键作用，能够检测故障并优雅地恢复，以维持持续运行。

得益于提供的 API 和配置选项，在 Orleans 中搭建带有 Silo 的本地集群非常简单。开发人员可以快速创建本地集群环境用于测试和开发，从而专注于编写应用逻辑，而不会被分布式系统的复杂性所困扰。

今天就到这里。期待与你再次相见。
