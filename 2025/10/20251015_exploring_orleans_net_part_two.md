# [探索 Orleans.NET - 第二部分](https://medium.com/@ZaradarTR/exploring-orleans-net-part-ii-921bc00f358d)

随着 Orleans.NET 的出现，开发可扩展、有弹性的分布式应用程序变得更加容易。在这篇后续文章中，我们将探索 Mardi Gras .NET 2.0 <https://github.com/be-heroes/mardi-gras-dotnet> 的新功能，以研究改进的架构如何利用 Orleans.NET 来增强分布式事件管理。我们还将深入代码，重点介绍 Orleans 的一些关键组件，这些组件使其成为处理复杂、可扩展系统的强大工具。

## 从 Mardi Gras .NET 1.0 到 2.0 的演进

Mardi Gras .NET 1.0 是一个介绍性项目，展示了 Orleans.NET 的核心功能，专注于使用 grains 和 silos 的简单事件驱动系统。它很好地展示了 Orleans 如何抽象出分布式计算的复杂性，使开发人员能够专注于业务逻辑。

然而，Mardi Gras .NET 2.0 将此提升到了一个新的水平，优化了架构并提供了更高级的功能。内存流的增加和更好的状态管理使系统对于实际应用来说更加健壮和可靠。

## 探索 Mardi Gras .NET 2.0

Mardi Gras .NET 2.0 的核心是 Orleans 虚拟执行体模型，它使用 grain 来处理并发和状态管理。这些 grain（Orleans 称之为“虚拟执行体”）可以按需创建或销毁，其状态可以在分布式环境中持久化。

### Grain 与 Silo

在 Mardi Gras .NET 2.0 中，两个主要的 Orleans 组件——grain 和 silo——发挥着关键作用：

- Grain 是处理单个业务逻辑或状态的轻量级对象。 Person grain 管理个人信息，如姓名和能量，模拟进食、饮酒和聚会等互动。
- Silo 是 grain 的执行环境。它们允许 grain 在分布式系统中运行，抽象化了节点间通信和负载均衡的复杂性。

Orleans 对 grain 和 silo 的动态管理确保即使节点发生故障，系统也能通过在剩余的 silo 间重新分配 grain 来优雅地恢复。

![Grain 与 Silo](https://miro.medium.com/v2/resize:fit:1370/format:webp/0*2fNKncqR8eMAg4Lq.png)

### v2.0 Person Grain

Mardi Gras .NET 2.0 中的 Person grain 继承自 Grain\<PersonGrainState\> 并实现了 IPerson 接口。这种设计使其能够封装行为（如吃喝）和状态（如能量等级）。

以下是 2.0 版本中 Person grain 的代码：

```csharp
using Domain.Interfaces;
using Orleans.Streams;

namespace Domain.Grains;

public class Person(ILogger<Person> logger) : Grain<PersonGrainState>, IPerson
{
    private readonly ILogger<Person> _logger = logger;
    private IAsyncStream<string> _eventStream = default!;

    public Task<string> GetNameAsync()
    {
        return Task.FromResult(State.Name);
    }

    public async Task SetNameAsync(string name)
    {
        State.Name = name;
        await WriteStateAsync();
    }

    public async Task EatAsync()
    {
        _logger.LogInformation($"{State.Name} is eating.");
        State.Energy = Math.Min(State.Energy + 10, 100);
        EnsureEnergyWithinBounds();
        
        await WriteStateAsync();
    }

    public async Task DrinkAsync()
    {
        if (State.Energy < 5) throw new InvalidOperationException($"{State.Name} does not have enough energy to drink.");
        _logger.LogInformation($"{State.Name} is drinking.");        
        State.Energy -= 5;
        EnsureEnergyWithinBounds();
        
        await WriteStateAsync();
    }

    public async Task PartyAsync()
    {
        if (State.Energy < 20) throw new InvalidOperationException($"{State.Name} does not have enough energy to party.");
        _logger.LogInformation($"{State.Name} is partying.");
        State.Energy -= 20;
        EnsureEnergyWithinBounds();
        
        await WriteStateAsync();
    }

    public async Task SleepAsync()
    {
        _logger.LogInformation($"{State.Name} is sleeping.");
        State.Energy = Math.Min(State.Energy + 30, 100);
        EnsureEnergyWithinBounds();
        
        await WriteStateAsync();
    }

    public Task<string> GetStatusAsync()
    {
        return Task.FromResult(ToString());
    }

    public override async Task OnActivateAsync(CancellationToken cancellationToken = default)
    {
        var streamProvider = GrainStreamingExtensions.GetStreamProvider(this, "Default");
        
        _eventStream = streamProvider.GetStream<string>(this.GetPrimaryKey());
        await SubscribeToEventsAsync();
        await base.OnActivateAsync(cancellationToken);
    }

    public async Task SubscribeToEventsAsync()
    {
        await _eventStream.SubscribeAsync((data, token) =>
        {
            return HandleEventAsync(data);
        });
    }

    public Task HandleEventAsync(string eventMessage)
    {
        _logger.LogInformation($"Received event: {eventMessage}");
        return Task.CompletedTask;
    }

    public override string ToString()
    {
        return $"{State.Name} (Energy: {State.Energy})";
    }

    private void EnsureEnergyWithinBounds()
    {
        if (State.Energy > 100) State.Energy = 100;
        if (State.Energy < 0) State.Energy = 0;
    }
}
```

主要特性：

- 状态持久化：grain 的状态存储在 PersonGrainState 中，由 Orleans 自动管理。 WriteStateAsync() 会异步提交更改。
- 事件流：Grain 从 Silo 激活一个事件流，以与其他 Grain 进行通信。

### Mardi Gras .NET 2.0 中的 Silo 设置

为 Orleans Silo 设置本地集群非常简单。以下代码演示了 Mardi Gras .NET 2.0 如何使用 localhost 集群、内存 grain 存储、内存流和控制台日志来配置 silo：

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

IHostBuilder builder = Host.CreateDefaultBuilder(args)
    .UseOrleans(silo =>
    {
        silo.UseLocalhostClustering()
            .AddMemoryGrainStorage("Default")
            .AddMemoryStreams("Default")
            .ConfigureLogging(logging => logging.AddConsole());
    })
    .UseConsoleLifetime();

using IHost host = builder.Build();

await host.RunAsync();
```

主要特性：

- 内存流：Mardi Gras .NET 2.0 引入了内存流，用于高效的 grain 间通信。这使得 grain 能够异步交换消息，提高响应能力和扩展能力。在 Orleans 中，流是处理高频更新或通知的绝佳机制。例如，代表事件的 grain 可以向其他 grain 广播状态变化（例如，参与者的行为）。
- 状态管理和持久化：Orleans 的自动状态管理简化了分布式数据的处理。例如，在 Person grain 中，人的姓名和能量是其状态的一部分，Orleans 管理这些数据在多个服务器上的持久化。在 Mardi Gras .NET 2.0 中， Person  grain 的状态使用 Orleans grain 存储机制进行持久化。无论一个人是在吃饭、喝酒还是聚会，状态都会在交互过程中被安全地存储和检索。

![Grain 交互](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*gGz_oPf7Y7jlJYcm.png)

## Grain 交互：事件与参与者

Mardi Gras .NET 的一个主要特性是 Event 与参与者（由 grains 建模）之间的交互。事件 grain 管理参与者列表（每个参与者由一个 Person grain 表示）。参与者可以与事件交互，系统确保状态变更得到高效处理。

以下是事件代码，用于说明参与者如何加入并交互：

```csharp
using Domain.Interfaces;
namespace Domain.Grains;

public class Event(ILogger<Event> logger) : Grain<EventGrainState>, IEvent
{
    private readonly ILogger _logger = logger;
    
    public IEnumerable<IPerson> Attendees => State.Attendees.AsReadOnly();

    public async ValueTask<bool> AddAttendeeAsync(IPerson attendee)
    {
        State.Attendees.Add(attendee);
        
        await WriteStateAsync();
        _logger.LogInformation($"{await attendee.GetNameAsync()} has joined the event: {State.Name}");
        return true;
    }

    public Task<string> GetNameAsync()
    {
        return Task.FromResult(State.Name);
    }

    public async Task SetNameAsync(string value)
    {        
        State.Name = value;
        await WriteStateAsync();
    }

    public async Task StartEventAsync()
    {
        _logger.LogInformation($"Event {State.Name} is starting!");
        var drinkTasks = State.Attendees.Select(async attendee =>
        {
            await attendee.DrinkAsync();
            _logger.LogInformation($"{await attendee.GetNameAsync()} had his welcome drink and is still sober.");
        });
        await Task.WhenAll(drinkTasks);
    }
}
```

能够在多个 silo 之间扩展此事件——同时确保所有参与者的状态被并发管理——展示了 Orleans.NET 虚拟 actor 模型的强大之处。

## 总结

Mardi Gras .NET 2.0 展示了 Orleans.NET 如何赋能开发者构建分布式应用程序，这些应用能够轻松跨多台机器扩展、有效管理状态并优雅处理故障。通过抽象化分布式系统的复杂性，Orleans 使开发者能够专注于真正重要的部分——交付可靠的业务逻辑。

如果你 正在构建云原生应用程序，Orleans.NET 提供了一个直观且强大的框架来高效管理分布式系统，正如 Mardi Gras .NET 2.0 所展示的那样。无论你 是在本地托管还是在云端托管，Orleans.NET 都能在幕后处理复杂性的同时扩展你 的应用程序。

总之，从 Mardi Gras .NET 1.0 到 2.0 的转变展示了使用 Orleans.NET 进行分布式系统设计的演进。凭借使状态管理和并发性更加简单的特性，Orleans 继续证明自己是软件开发领域中的一款强大工具。
