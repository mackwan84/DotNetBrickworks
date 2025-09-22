# [为 .NET 项目选择正确的项目架构](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)

选择合适的项目结构是 .NET 开发团队最具影响力的决策之一。它影响着从入职速度、代码可维护性到功能交付速度的方方面面。

在现代软件开发中，有 3 种流行的项目结构方法：

- N 层架构（控制器-服务-仓储）
- 整洁架构
- 垂直切片架构

在过去的几年里，我使用这三种方法构建和扩展了.NET 解决方案。今天，我将剖析真正的权衡，分享我在实践中看到的陷阱，并展示我的个人偏好，以帮助团队同时兼顾开发速度和可维护性。

在本文中，我们将探讨每种方法的优缺点：

- N 层架构
- 整洁架构
- 垂直切片架构
- 将整洁架构与垂直切片相结合
- 个人认为的最佳架构

闲话少说，让我们开始吧！

## N 层架构

N 层方法仍然是大多数代码库中的默认选择——从简单到企业级 .NET 应用程序都广泛采用。

它围绕将职责分离到逻辑层来构建，最常见的是：

- 表示层：控制器、API 端点或 UI 组件
- 业务逻辑层（服务）：封装业务规则的应用服务
- 数据访问层（仓储）：数据持久化的抽象

这种结构易于理解，被广泛教授，几乎所有 .NET 开发者都熟悉。通常可以看到项目按以下方式组织：

- Controllers
- Services
- Repositories
- Models

![N 层架构](https://antondevtips.com/media/code_screenshots/architecture/nlayered-ca-vsa/img_4.png)

N 层架构如此受欢迎的主要原因是它易于理解。如果你加入一个新团队或项目，你会很快知道在哪里放置新代码，这使得开发人员的入职变得容易。

它强制实现了基本的关注点分离，对于中小型 CRUD 应用程序来说，这是一个简单的默认选择。

使用 N 层架构的典型 .NET Web API 项目可能如下所示：

```csharp
[ApiController]
[Route("api/shipments")]
public class ShipmentsController : ControllerBase
{
    private readonly IShipmentService _service;
    
    public ShipmentsController(IShipmentService service)
    {
        _service = service;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var shipment = await _service.GetShipmentByIdAsync(id);
        return Ok(shipment);
    }
}
```

```csharp
public class ShipmentService : IShipmentService
{
    private readonly IShipmentRepository _repository;
    
    public ShipmentService(IShipmentRepository repository)
    {
        _repository = repository;
    }

    public async Task<ShipmentDto> GetShipmentByIdAsync(int id)
    {
        return await _repository.GetByIdAsync(id);
    }
}
```

```csharp
public class ShipmentRepository : IShipmentRepository
{
    private readonly ShipmentDbContext _dbContext;
    
    public ShipmentRepository(ShipmentDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<ShipmentDto> GetByIdAsync(int id)
    {
        return await _dbContext.Shipments
            .Where(s => s.Id == id)
            .Select(s => new ShipmentDto { Number = s.Number, OrderId = s.OrderId })
            .FirstOrDefaultAsync();
    }
}
```

虽然这种方法看似简单，但在你的代码库中可能会遇到一些潜在问题。

### 臃肿的控制器和臃肿的服务

N 层架构最常见的问题之一是，随着业务需求的发展，控制器和服务往往会迅速膨胀。

最初只是一个有 4 个方法的简单 CRUD 操作，很快就变成了一个处理数十种职责的大型类。

以下是经过几个月或几年开发后，典型的 ShipmentsController 的样子：

```csharp
[ApiController]
[Route("api/[controller]")]
public class ShipmentsController : ControllerBase
{
    private readonly IShipmentService _shipmentService;
    
    public ShipmentsController(IShipmentService shipmentService)
    {
        _shipmentService = shipmentService;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetShipment(int id);
    
    [HttpGet("user/{userId}")]
    public async Task<IActionResult> GetShipmentsByUser(int userId);
    
    [HttpGet("date-range")]
    public async Task<IActionResult> GetShipmentsByDateRange(DateTime from, DateTime to);
    
    [HttpPost]
    public async Task<IActionResult> CreateShipment(CreateShipmentRequest request);
    
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateShipment(int id, UpdateShipmentRequest request);
    
    [HttpPatch("{id}/status")]
    public async Task<IActionResult> UpdateShipmentStatus(int id, ShipmentStatus status);
    
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteShipment(int id);
    
    [HttpPost("{id}/track")]
    public async Task<IActionResult> TrackShipment(int id);
    
    [HttpPost("{id}/cancel")]
    public async Task<IActionResult> CancelShipment(int id);
    
    [HttpPost("{id}/approve")]
    public async Task<IActionResult> ApproveShipment(int id);
}
```

是的，你可以将这些端点分离到多个控制器中，但对开发者来说，添加一个新方法比提取一个新控制器感觉更容易。服务和仓库类的情况可能会比这更糟糕。

### 过多的小型服务和存储库

随着你的领域不断增长，你将面临一个关键决策：是否应该为每个实体创建服务和存储库？

当处理 Shipments 、 ShipmentItems 和 Orders 时，遵循传统的 N 层架构方法会导致：

```csharp
public interface IShipmentRepository
{
    Task<ShipmentDto> GetByIdAsync(int id);
    Task<IEnumerable<ShipmentDto>> GetAllAsync();
    // ...
}

public interface IShipmentItemRepository
{
    Task<ShipmentItemDto> GetByIdAsync(int id);
    Task<IEnumerable<ShipmentItemDto>> GetByShipmentIdAsync(int shipmentId);
    // ...
}

public interface IOrderRepository
{
    Task<OrderDto> GetByIdAsync(int id);
    Task<IEnumerable<OrderDto>> GetByUserIdAsync(int userId);
    // ...
}
```

但当你需要同时获取相关实体时会发生什么？例如：

- 获取包含所有项目的发货信息
- 获取订单及其关联的发货信息
- 获取加载多个实体的发货历史数据

**这些跨实体方法应该放在哪里？**

许多开发者最终拥有大量功能不足的服务和仓储。当你实现一个新功能时，你开始思考应该在三个仓储中的哪一个里添加新方法。

### 业务逻辑薄弱且难以测试的解决方案

业务规则通常分散在多个服务类中，这使得识别和理解它们变得具有挑战性。

在 N 层架构中，没有严格的规则；例如，你可以绕过任何层，直接在控制器中使用存储库。此外，你可以省略接口，直接将实现传递给各层，这会导致代码难以测试。

结果呢？

你最终会得到一些棘手的解决方案：

- 臃肿的仓储，它们对其他实体了解得太多
- 太多只有 1-2 个方法的小仓储
- 协调多个存储库的服务方法，导致 N+1 查询问题
- 当不同服务需要类似的跨实体操作时，会出现重复逻辑

这就是传统的 N 层架构开始崩溃的地方。控制器和服务变得越来越大，存储库要么做得不够，要么试图做得太多，随着业务需求的变化，你的代码变得更难理解和修改。

虽然这种模式对于小型或简单项目来说并不"糟糕"，但对于希望更快地进行变更、拥有更清晰代码和真正模块化的团队来说，它很少是最佳选择。

现在许多.NET 团队正在构建更复杂的系统，如模块化单体、领域驱动设计（DDD）或微服务。根据我的经验，我已经使用 N 层架构构建了许多项目，但我意识到它正在阻碍我的发展。

几年前，我开始在我的项目中采用**整洁架构**

## 整洁架构

我知道有些团队在微服务中使用 N 层架构进行 CRUD 操作，并在编写高质量代码和遵循既定规则方面保持高度纪律性。但对于大多数团队来说，情况并非如此。

整洁架构开箱即用地为你提供这种纪律性。

整洁架构旨在将应用程序的关注点分离到不同的层中，促进高内聚和低耦合。

它包含以下层：

- 领域层：包含核心业务对象，如实体。
- 应用层：实现业务用例（类似于 N 层架构中的服务层）。
- 基础设施层：实现外部依赖，如数据库、缓存、消息队列、身份验证提供程序等。
- 表示层：与外部世界的接口实现，如 WebApi、gRPC、GraphQL、MVC 等。

![整洁架构](https://antondevtips.com/media/code_screenshots/architecture/clean-architecture-with-vertical-slices/ca_with_vsa_4.png)

通常我们会看到解决方案是这样组织的：

- Domain  领域
- Application  应用
  - /Queries  /查询
  - /QueryHandlers  /查询处理器
  - /Commands  /命令
  - /CommandHandlers  /命令处理器
- Infrastructure  基础设施层
- Presentation  表示层

**整洁架构解决了 N 层架构的许多缺点：**

- 关注点分离：整洁架构强制在应用程序的不同层之间实现清晰分离。所有层都指向内部：你不能从领域层调用应用程序用例。这使得代码库更易于维护和理解。
- 可测试性：通过将业务逻辑与基础设施和用户界面分离，整洁架构使得编写单元测试变得更加容易。应用程序的核心（用例和实体）可以在不依赖外部依赖项和具体实现的情况下进行测试。
- 灵活性：整洁架构允许你更改技术栈（例如，从一个数据库提供商切换到另一个），而对核心业务逻辑的影响最小。这种灵活性是通过将基础设施关注点抽象到核心应用程序所依赖的接口后面来实现的。
- 代码可重用性：通过将核心业务逻辑与实现细节解耦，整洁架构鼓励在不同项目或同一项目的不同层之间重用代码。

根据我自己的经验，我可以告诉你整洁架构具有高度适应性，能够在保持相同代码质量的同时应对不断变化的业务需求。而在 N 层架构中，面对不断演进的变更，要维持相同水平的代码质量可能会很有挑战性。

典型的清洁架构预期在 API 端点中使用 MediatR 或手动处理程序：

```csharp
public class CreateShipmentEndpoint : IEndpoint
{
    public void MapEndpoint(WebApplication app)
    {
        app.MapPost("/api/shipments", Handle);
    }
    
    private static async Task<IResult> Handle(
        [FromBody] CreateShipmentRequest request,
        IValidator<CreateShipmentRequest> validator,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        var validationResult = await validator.ValidateAsync(request, cancellationToken);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }
                
        var command = request.MapToCommand();

        var response = await mediator.Send(command, cancellationToken);
        if (response.IsError)
        {
            return response.Errors.ToProblem();
        }

        return Results.Ok(response.Value);
    }
}
```

API 端点位于表示层项目内部，并调用应用层处理程序（或服务）。

在这种情况下， CreateShipmentCommandHandler 使用 IShipmentRepository 并且对基础设施层中的实际 ShipmentRepository 实现一无所知：

```csharp
internal sealed class CreateShipmentCommandHandler(
    IShipmentRepository repository,
    IUnitOfWork unitOfWork,
    ILogger<CreateShipmentCommandHandler> logger)
    : IRequestHandler<CreateShipmentCommand, ErrorOr<ShipmentResponse>>
{
    public async Task<ErrorOr<ShipmentResponse>> Handle(
        CreateShipmentCommand request,
        CancellationToken cancellationToken)
    {
        var shipmentAlreadyExists = await repository.ExistsByOrderIdAsync(request.OrderId, cancellationToken);
        if (shipmentAlreadyExists)
        {
            logger.LogInformation("Shipment for order '{OrderId}' is already created", request.OrderId);
            return Error.Conflict($"Shipment for order '{request.OrderId}' is already created");
        }

        var shipmentNumber = new Faker().Commerce.Ean8();
        var shipment = request.MapToShipment(shipmentNumber);

        await repository.AddAsync(shipment, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);

        logger.LogInformation("Created shipment: {@Shipment}", shipment);

        var response = shipment.MapToResponse();
        return response;
    }
}
```

### 实用清洁架构

好的，现在我们有了明确的指向内部的层级分离，但我们仍然面临着与 N 层架构中相同的仓储问题。

随着时间的推移，清洁架构已经演变成一种更实用的方法：开发人员同意他们可以在应用程序用例中使用 EF Core。

是的，你没听错：在命令处理程序中直接使用 EF Core，而不创建仓储。

但这难道不会破坏清洁架构提供的所有好处吗？不完全是。让我来解释。

首先，EF Core 的 DbContext 已经实现了仓储和工作单元模式，正如官方 DbContext 代码摘要中所述。当我们在 EF Core 之上创建仓储时，我们是在一个抽象之上又创建了另一个抽象，导致过度工程化的解决方案。

其次，你在生产环境中更换过多少次数据库？

在 99% 的情况下，你不需要更换数据库。然而，即使更换，也不仅仅是将 EF Core 更改为 MongoDB 这么简单。

如果你切换到一个完全不同的数据库，可能需要完全重写现有应用程序，因为数据访问模式可能会发生显著变化。

另一方面，当你从一个 SQL 数据库切换到另一个（例如，Postgres → SQL Server）时，EF Core 中 95% 的代码不需要更改。

那么可测试性以及在不同类中重复 EF 调用的问题呢？

对于单元测试，你可以使用内存 DbContext，而使用集成测试来测试数据库逻辑则更好。为了消除重复的 EF Core 查询，你可以使用 Specification 模式。最终，作为一种权衡，你可以为少数查询创建 Repository 以避免代码重复。

编写软件没有唯一正确的方法；你需要根据每个特定项目和情况选择最适合的方案。

这就是为什么在应用程序用例中直接使用 EF Core 是一种利大于弊的权衡。

当我们摆脱存储库时， CreateShipmentCommandHandler 的变化如下：

```csharp
internal sealed class CreateShipmentCommandHandler(
    ShipmentsDbContext context,
    ILogger<CreateShipmentCommandHandler> logger)
    : IRequestHandler<CreateShipmentCommand, ErrorOr<ShipmentResponse>>
{
    public async Task<ErrorOr<ShipmentResponse>> Handle(
        CreateShipmentCommand request,
        CancellationToken cancellationToken)
    {
        var shipmentAlreadyExists = await context.Shipments.AnyAsync(x => x.OrderId == request.OrderId, cancellationToken);
        if (shipmentAlreadyExists)
        {
            logger.LogInformation("Shipment for order '{OrderId}' is already created", request.OrderId);
            return Error.Conflict($"Shipment for order '{request.OrderId}' is already created");
        }

        var shipmentNumber = new Faker().Commerce.Ean8();
        var shipment = request.MapToShipment(shipmentNumber);

        await context.Shipments.AddAsync(shipment, cancellationToken);
        await context.SaveChangesAsync(cancellationToken);

        logger.LogInformation("Created shipment: {@Shipment}", shipment);

        var response = shipment.MapToResponse();
        return response;
    }
}
```

### 整洁架构和富领域模型

好的，我们现在有了严格的规则（你可以通过架构测试进一步加强它们），但业务逻辑呢？业务逻辑通常分散并重复在各个服务类中，导致解决方案难以维护。这主要是由于使用了贫血的领域实体。

让我们来探索这样一个贫血模型的例子：

```csharp
public class Shipment
{
    public Guid Id { get; set; }
    public string Number { get; set; }
    public string OrderId { get; set; }
    public Address Address { get; set; }
    public string Carrier { get; set; }
    public string ReceiverEmail { get; set; }
    public ShipmentStatus Status { get; set; }
    public List<ShipmentItem> Items { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}
```

当你查看这个实体时，你只能看到具有公共 getter 和 setter 的属性。这个实体没有提供关于货运状态如何变更或添加货运项目时应执行哪些检查的任何信息。

由于所有属性都通过公共设置器公开，没有什么能阻止开发人员从另一个类更新该属性，从而绕过应用层中的任何检查。

为了解决这个问题，随着开发人员开始使用富领域模型，整洁架构应运而生。富领域模型强制将业务逻辑封装到领域实体中。

```csharp
public class Shipment
{
    private readonly List<ShipmentItem> _items = [];
    
    public Guid Id { get; private set; }
    public string Number { get; private set; }
    public string OrderId { get; private set; }
    public Address Address { get; private set; }
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
        string orderId,
        Address address,
        string carrier,
        string receiverEmail,
        List<ShipmentItem> items)
    {
        var shipment = new Shipment { ... };
        shipment.AddItems(items);
        
        return shipment;
    }
    
    public void AddItem(ShipmentItem item) { ... };
    public void AddItems(List<ShipmentItem> items) { ... };
    public void RemoveItem(ShipmentItem item) { ... };
    public void UpdateAddress(Address newAddress) { ... };
    
    public ErrorOr<Success> Process()
    {
        if (Status is not ShipmentStatus.Created)
        {
            return Error.Validation("Can only update to Processing from Created status");
        }
        
        Status = ShipmentStatus.Processing;
        UpdatedAt = DateTime.UtcNow;

        return Result.Success;
    }

    public ErrorOr<Success> Dispatch()
    {
        if (Status is not ShipmentStatus.Processing)
        {
            return Error.Validation("Can only update to Dispatched from Processing status");
        }
        
        Status = ShipmentStatus.Dispatched;
        UpdatedAt = DateTime.UtcNow;
        
        return Result.Success;
    }

    public ErrorOr<Success> Transit() { ... };
    public ErrorOr<Success> Deliver() { ... };
    public ErrorOr<Success> Receive() { ... };
    public ErrorOr<Success> Cancel() { ... };
}
```

现在， Shipment 类不会为其属性公开公共设置器；相反，你使用静态工厂方法 Create 来创建 Shipment 实例。通过这种方式，你可以将任何逻辑和验证封装在此方法内，确保创建实体的单一且正确的方式。

此外， Shipment 类提供了更新其状态的方法，包括 AddItem、Process、Dispatch、Cancel 等。通过这种方式，我们将逻辑封装在单一位置，避免业务规则分散和重复于不同类中。

### 整洁架构和功能归类

传统方法按照技术关注点组织代码——将控制器、查询处理器、服务和仓库分离到不同的文件夹中。然而，开发人员很快意识到，这使得理解和修改功能变得困难，因为相关代码分散在多个文件夹中。

清洁架构自然地从技术文件夹演变为按照功能归类文件夹。

向功能文件夹的演变将与特定用例相关的所有代码集中在一个地方。不再需要在实现 CreateShipment 用例时在 /Controllers 、 /QueryHandlers 、 /CommandHandlers 和 /Repositories 之间导航；现在你有一个 /Features/Shipments/CreateShipment 文件夹，包含了该功能所需的一切。

这使得代码库更直观地导航，并且无需在多个项目和 5-7 个文件之间跳转来实现单个功能。

虽然整洁架构有很多优点，但它并不是万能的。

**以下是整洁架构的缺点：**

- 复杂性：整洁架构引入了多层和抽象，这可能会增加代码库的复杂性，特别是对于小型项目。如果将这种架构不必要地应用于简单应用程序，开发人员可能会感到不知所措。
- 开销：关注点分离和接口的使用可能导致额外的模板代码，这可能会减慢开发过程。在较小项目中，这种开销可能特别明显，因为整洁架构的好处可能不那么显著。
- 学习曲线：对于不熟悉整洁架构的开发人员来说，存在陡峭的学习曲线。理解其原则并正确应用它们可能需要时间，特别是对于那些刚接触软件架构模式的人来说。
- 初始设置时间：从头开始搭建一个整洁架构项目需要仔细的规划和组织。与 N 层架构相比，初始设置时间可能更长。

现在，让我们探索一种新的、非常流行的代码结构方法：垂直切片架构。

## 垂直切片架构

垂直切片架构是当今组织项目的一种极为流行的方式。它追求在切片（功能）内实现高内聚，而在切片之间保持松耦合。

它通过功能而非技术层来构建应用程序。每个切片封装了特定功能的所有方面，包括 API、业务逻辑和数据访问。

![垂直切片架构](https://antondevtips.com/media/code_screenshots/architecture/clean-architecture-with-vertical-slices/ca_with_vsa_5.png)

以下是如何使用垂直切片架构组织项目的一个示例：

![垂直切片架构目录结构](https://antondevtips.com/media/code_screenshots/architecture/nlayered-ca-vsa/img_1.png)

每个文件夹代表一个功能（用例），每个功能可以由一个或多个文件组成。对于只有少量端点的简单 CRUD 服务，直接将 DbContext 注入到 API 端点中完全足够。

对于更复杂的项目，我会为每个用例创建处理器。以前我使用 MediatR 命令和查询来实现这一点，然而，我现在倾向于手动创建自己的处理器。

**垂直切片具有以下优势：**

- 功能聚焦：变更被隔离在特定功能中，减少了意外副作用的风险。
- 可扩展性：通过允许其他开发人员和团队独立处理不同功能，更容易扩展开发。
- 灵活性：允许在每个切片中根据需要使用不同的技术或方法。
- 可维护性：在解决方案中更容易导航、理解和维护，因为功能的所有方面都包含在单个切片中。
- 降低耦合：最小化不同切片之间的依赖关系。

**以下是垂直切片架构的缺点：**

- 潜在的代码重复：不同切片间可能存在代码重复。
- 一致性：确保各切片间的一致性并管理横切关注点（如错误处理、日志记录、验证）需要仔细规划。
- 大量类和文件：大型应用程序可能包含大量垂直切片，每个切片又包含多个小型类。

对于前两个缺点，你可以通过精心设计架构来应对。例如，你可以将通用功能提取到独立的类中。

要管理横切关注点（如错误处理、日志记录和验证），你可以使用 ASP.NET Core 中间件。一个结构良好的文件夹可以解决第三个缺点。

一方面，整洁架构为应用程序的不同层次提供了清晰的分离。但另一方面，你需要浏览多个项目来探索单个用例的实现。

整洁架构最大的优点之一是它为你的应用程序提供了以领域为中心的设计，显著简化了复杂领域和项目的开发。

另一方面，垂直切片架构允许你以一种提供快速导航和开发的方式组织代码。单个用例的实现集中在一个地方。

如果我们能够取两者之长，将整洁架构与垂直切片架构相结合，会怎么样呢？

## 将整洁架构与垂直切片架构相结合

我相信，带有功能文件夹的整洁架构的自然演变，已经使其转变为垂直切片架构。原始的整洁架构并不总是将解决方案分离为项目，而是关注于类及其关系。

主要点是内层的类不能调用外层的类。使用垂直切片架构，你可以实现相同的效果，但需要更少的项目。

我发现将整洁架构与 垂直切片架构相结合是复杂应用程序的一种优秀架构设计。在小型应用程序或没有复杂业务逻辑的应用程序中，你可以在不使用整洁架构的情况下使用垂直切片。

作为核心，我使用整洁架构的层次结构，并将它们与垂直切片相结合。

以下是各层次的修改方式：

- 领域：包含核心业务对象，如实体（保持不变）。
- 基础设施：实现外部依赖项，如数据库、缓存、消息队列、身份验证提供程序等（保持不变）。
- 应用程序层和表示层与垂直切片相结合。

![应用程序层和表示层与垂直切片相结合](https://antondevtips.com/media/code_screenshots/architecture/clean-architecture-with-vertical-slices/ca_with_vsa_6.png)

这样的项目结构看起来是这样的：

![结合后的目录结构](https://antondevtips.com/media/code_screenshots/architecture/modular-monolith-vertical-slices/img_1.png)

在基础设施项目中，我喜欢放置外部集成的实现，如数据库、缓存和身份验证。如果你的项目不需要实现仓储或其他外部集成，你可以省略基础设施项目。

CreateShipment 用例的实现实际上与整洁架构解决方案中的相同：

```csharp
public class CreateShipmentEndpoint : IEndpoint
{
    public void MapEndpoint(WebApplication app)
    {
        app.MapPost("/api/shipments", Handle);
    }
    
    private static async Task<IResult> Handle(
        [FromBody] CreateShipmentRequest request,
        IValidator<CreateShipmentRequest> validator,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        var validationResult = await validator.ValidateAsync(request, cancellationToken);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }
                
        var command = request.MapToCommand();

        var response = await mediator.Send(command, cancellationToken);
        if (response.IsError)
        {
            return response.Errors.ToProblem();
        }

        return Results.Ok(response.Value);
    }
}
```

垂直切片架构与清洁架构一样，需要新开发人员一些时间来熟悉，但能显著缩短他们理解实际用例和业务领域所需的时间。

## 2025 年使用的最佳架构

在探索了 N 层架构、整洁架构和垂直切片架构之后，你可能会想：下一个项目应该选择哪种架构？答案取决于你的团队规模、项目复杂性和业务需求。

### 何时使用 N 层架构

它最适合：

- 小型团队（1-3 名开发人员）
- 简单的 CRUD 应用程序
- 业务逻辑简单的项目
- 当你需要快速实现功能且开发人员已经熟悉 N 层架构时

选择 N 层架构的情况：

- 你的应用程序主要是数据驱动的，业务规则极少
- 你的团队中有初级开发人员，他们需要一个熟悉且易于理解的结构
- 项目范围明确且不太可能发生重大变化
- 你正在构建原型或概念验证应用程序

### 何时使用整洁架构

它最适合：

- 中大型团队（4-10 名开发人员）
- 具有丰富逻辑的复杂业务领域
- 需要适应不断变化需求的长期项目
- 需要高度可测试性的应用程序

在以下情况下选择整洁架构：

- 你有需要隔离和保护的复杂业务规则
- 你的团队具备架构模式方面的经验
- 你需要与多个外部系统集成
- 该项目将需要维护和扩展数年之久
- 构建单体或模块化单体应用程序

### 何时使用垂直切片架构

它最适合：

- 任何重视快速开发速度的团队
- 以功能为中心的开发工作流
- 具有许多独立功能的应用程序
- 不同功能可能使用不同技术的项目

在以下情况下选择垂直切片架构：

- 你希望最小化实现新功能所需的时间
- 你的团队以功能为中心进行冲刺开发
- 你需要让新开发人员快速熟悉你的业务领域
- 你应用程序的不同部分具有不同的复杂度级别

### 何时使用整洁架构+垂直切片

它最适合：

- 中型或大型应用程序
- 希望兼具结构和灵活性的团队
- 具有丰富业务领域和众多功能的项目
- 需要在代码和团队规模两方面都能扩展的应用程序

在以下情况下选择整洁架构+垂直切片：

- 你有一个复杂的领域，可以从整洁结构中受益
- 你想要垂直切片架构带来的开发速度优势
- 你的团队对这两种架构模式都有经验
- 你正在构建一个会随时间显著增长的系统
- 构建单体或模块化单体应用程序

## 总结

对于 2025 年开始的大多数新项目，我推荐结合垂直切片的整洁架构。

这种方法为你提供：

- 需要时的结构：整洁架构为复杂业务逻辑提供坚实基础
- 想要时的速度：垂直切片允许快速功能开发
- 未来的灵活性：易于添加新功能和更改现有功能
- 团队可扩展性：不同团队可以独立开发不同功能

关键在于选择一种既符合当前团队技能、项目复杂性和业务需求，又兼顾未来发展的方法。

从你当前情况最合理的方式开始：

- 正在处理现有项目？在遇到添加新功能需要耗费过多精力和时间的情况之前，无需将 N 层架构更改为其他任何架构
- 构建模块化单体？选择整洁架构 + 垂直切片
- 想要最快的开发速度？选择纯垂直切片架构

今天就到这里。期待与你再次相见。
