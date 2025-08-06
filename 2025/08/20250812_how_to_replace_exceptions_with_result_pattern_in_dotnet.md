# [如何在.NET 中用 Result 模式替代异常](https://antondevtips.com/blog/how-to-replace-exceptions-with-result-pattern-in-dotnet)

在现代软件开发中，优雅地处理错误和异常情况对于构建健壮的应用程序至关重要。虽然异常是.NET 中常见的错误处理机制，但它们可能会带来性能开销并使代码流程复杂化。

今天我们将探讨如何在 .NET 中使用 **Result 模式**替代异常，从而提高代码的可读性、可维护性和性能。

## .NET 中的异常处理简介

异常处理是 .NET 编程中的一个基本概念，允许开发人员优雅地管理运行时错误。典型的方法是使用 try、catch 和 finally 块来捕获和处理异常。

让我们探索一个创建货运并使用异常进行控制流的应用程序：

```csharp
public async Task<ShipmentResponse> CreateAsync(
    CreateShipmentCommand request,
    CancellationToken cancellationToken)
{
    var shipmentAlreadyExists = await context.Shipments
        .Where(s => s.OrderId == request.OrderId)
        .AnyAsync(cancellationToken);

    if (shipmentAlreadyExists)
    {
        throw new ShipmentAlreadyExistsException(request.OrderId);
    }

    var shipment = request.MapToShipment(shipmentNumber);
    
    context.Shipments.Add(shipment);
    await context.SaveChangesAsync(cancellationToken);

    return shipment.MapToResponse();
}
```

如果数据库中已存在发货记录，则会抛出 ShipmentAlreadyExistsException 。在最小 API 端点中，此异常的处理方式如下：

```csharp
public void MapEndpoint(WebApplication app)
{
    app.MapPost("/api/v1/shipments", Handle);
}

private static async Task<IResult> Handle(
    [FromBody] CreateShipmentRequest request,
    IShipmentService service,
    CancellationToken cancellationToken)
{
    try
    {
        var command = request.MapToCommand();
        var response = await service.CreateAsync(command, cancellationToken);
        return Results.Ok(response);
    }
    catch (ShipmentAlreadyExistsException ex)
    {
        return Results.Conflict(new { message = ex.Message });
    }
}
```

虽然这种方法可行，但它有以下缺点：

- 代码是不可预测的，通过查看 **IShipmentService.CreateAsync** 你无法确定该方法是否会抛出异常
- 你需要使用 catch 语句，并且无法再逐行从上到下阅读代码，你需要来回切换视图
- 异常在某些应用程序中可能导致性能问题，因为它们非常慢（在最近的.NET 9 中它们变得更快了，但仍然很慢）

请记住，异常是用于异常情况的。它们不是控制流的最佳选择。相反，我想向你展示一种使用 **Result 模式** 的更好方法。

## 理解 Result 模式

**Result 模式** 是一种设计模式，它封装了操作的结果，该结果可以是成功或失败。方法不会抛出异常，而是返回一个 Result 对象，指示操作是成功还是失败，以及任何相关数据或错误消息。

Result 对象由以下部分组成：

- IsSuccess/IsError：一个布尔值，用于指示操作是否成功。
- Value：操作成功时的结果值。
- Error：当操作失败时的错误消息或对象。

让我们来探索一个关于如何实现 Result 对象的简单示例：

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value)
    {
        return new Result<T>(true, value, null);
    }

    public static Result<T> Failure(string error)
    {
        return new Result<T>(false, default(T), error);
    }
}
```

这里我们定义了一个泛型 Result 类，它可以包含成功值或错误。要创建 Result 对象，你可以使用 **Success** 或 **Failure** 静态方法。

让我们使用 Result 模式重写之前的 Create Shipment 端点实现：

```csharp
public async Task<Result<ShipmentResponse>> CreateAsync(
    CreateShipmentCommand request,
    CancellationToken cancellationToken)
{
    var shipmentAlreadyExists = await context.Shipments
        .Where(s => s.OrderId == request.OrderId)
        .AnyAsync(cancellationToken);

    if (shipmentAlreadyExists)
    {
        return Result.Failure<ShipmentResponse>($"Shipment for order '{request.OrderId}' is already created");
    }

    var shipment = request.MapToShipment(shipmentNumber);
    
    context.Shipments.Add(shipment);
    await context.SaveChangesAsync(cancellationToken);

    var response = shipment.MapToResponse();
    return Result.Success(response);
}
```

这里的方法返回 Result\<ShipmentResponse\> ，它将 ShipmentResponse 包装在一个 Result 类中。

当数据库中已存在发货记录时，我们返回 **Result.Failure** 并附带相应消息。当请求成功时，我们返回 **Result.Success** 。

以下是在端点中处理结果对象的方法：

```csharp
public void MapEndpoint(WebApplication app)
{
    app.MapPost("/api/v1/shipments", Handle);
}

private static async Task<IResult> Handle(
    [FromBody] CreateShipmentRequest request,
    IShipmentService service,
    CancellationToken cancellationToken)
{
    var command = request.MapToCommand();
    var response = await service.CreateAsync(command, cancellationToken);
    
    return response.IsSuccess ? Results.Ok(response.Value) : Results.Conflict(response.Error);
}
```

你需要检查响应是成功还是失败，并返回适当的 HTTP 结果。现在代码看起来更加可预测，也更容易阅读，对吧？

当前的 Result 对象实现非常简化，在实际应用程序中，你需要其中包含更多功能。你可以花些时间为自己构建一个，并在所有项目中重用。或者你可以使用现成的 Nuget 包。

有很多实现了 **Result 模式** 的 Nuget 包，让我为你介绍几个最受欢迎的：

- [FluentResults](https://github.com/altmann/FluentResults)
- [CSharpFunctionalExtensions](https://github.com/vkhorikov/CSharpFunctionalExtensions)
- [error-or](https://github.com/amantinband/error-or)
- [Ardalis.Result](https://github.com/ardalis/Result)

我最喜欢的是 **error-or** ，让我们来探索一下。

## 使用 error-or 实现 Result 模式

正如该包的作者所述：Error-Or 是一个简单、流畅的错误或结果的区分联合。此库中的 Result 类被称为 ErrorOr\<T\> 。

以下是如何使用它进行控制流：

```csharp
public async Task<ErrorOr<ShipmentResponse>> Handle(
    CreateShipmentCommand request,
    CancellationToken cancellationToken)
{
    var shipmentAlreadyExists = await context.Shipments
        .Where(s => s.OrderId == request.OrderId)
        .AnyAsync(cancellationToken);

    if (shipmentAlreadyExists)
    {
        return Error.Conflict($"Shipment for order '{request.OrderId}' is already created");
    }

    var shipment = request.MapToShipment(shipmentNumber);

    context.Shipments.Add(shipment);
    await context.SaveChangesAsync(cancellationToken);

    return shipment.MapToResponse();
}
```

这里的返回类型是 ErrorOr\<ShipmentResponse\> ，表示是否返回错误或 ShipmentResponse 。

Error 类为以下类型提供了内置错误，并为每种**错误类型**提供了一个方法：

```csharp
public enum ErrorType
{
    Failure,
    Unexpected,
    Validation,
    Conflict,
    NotFound,
    Unauthorized,
    Forbidden,
}
```

该库允许你在需要时创建自定义错误。

在 API 端点中，你可以处理错误，类似于我们之前使用自定义 Result Object 所做的操作：

```csharp
public void MapEndpoint(WebApplication app)
{
    app.MapPost("/api/v2/shipments", Handle);
}

private static async Task<IResult> Handle(
    [FromBody] CreateShipmentRequest request,
    IShipmentService service,
    CancellationToken cancellationToken)
{
    var command = request.MapToCommand();

    var response = await mediator.Send(command, cancellationToken);
    if (response.IsError)
    {
        return Results.Conflict(response.Errors);
    }

    return Results.Ok(response.Value);
}
```

在使用 Error-Or 时，我喜欢创建一个静态方法 **ToProblem** ，该方法将错误转换为适当的状态码：

```csharp
public static class EndpointResultsExtensions
{
    public static IResult ToProblem(this List<Error> errors)
    {
        if (errors.Count == 0)
        {
            return Results.Problem();
        }

        return CreateProblem(errors);
    }

    private static IResult CreateProblem(List<Error> errors)
    {
        var statusCode = errors.First().Type switch
        {
            ErrorType.Conflict => StatusCodes.Status409Conflict,
            ErrorType.Validation => StatusCodes.Status400BadRequest,
            ErrorType.NotFound => StatusCodes.Status404NotFound,
            ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
            _ => StatusCodes.Status500InternalServerError
        };

        return Results.ValidationProblem(errors.ToDictionary(k => k.Code, v => new[] { v.Description }),
            statusCode: statusCode);
    }
}
```

这样，之前的代码将被转换为：

```csharp
var response = await mediator.Send(command, cancellationToken);
if (response.IsError)
{
    return response.Errors.ToProblem();
}

return Results.Ok(response.Value);
```

## 使用 Result 模式的优缺点

优点：

- 显式错误处理：调用者必须显式处理成功和失败情况。从方法签名可以明显看出，可能会返回错误。
- 提升性能：减少与异常相关的开销。
- 更好的测试：简化了单元测试，因为模拟结果对象比抛出和处理异常要容易得多。
- 安全性：结果对象应包含可向外部世界展示的信息。同时，你可以使用 Logger 或其他工具保存所有详细信息。

潜在缺点：

- 冗长性：与使用异常相比，可能会引入更多代码，因为你需要标记堆栈跟踪中的所有方法以返回 Result 对象
- 并非适用于所有情况：对于在正常操作期间不会发生的真正异常情况，异常处理仍然是合适的选择。

**Result 模式** 看起来很棒，但我们是否应该完全忘记异常？绝对不是！异常仍然有其用武之地。让我们来讨论一下。

## 何时使用异常

异常适用于异常情况，我认为在以下使用场景中它们可能是一个合适的选择：

- 全局异常处理
- 库代码
- 领域实体保护验证

让我们仔细看看这 3 种情况：

### 全局异常处理

在你的 asp.net core 应用程序中，你肯定需要处理异常。异常可能被抛出到任何地方：数据库访问、网络调用、I/O 操作、库等。

你需要准备好处理异常并优雅地处理它们。为此，我实现了从 .NET 8 开始可用的 IExceptionHandler ：

```csharp
internal sealed class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Exception occurred: {Message}", exception.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server error"
        };

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;

        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

### 库代码

在库中，通常在出现问题时会抛出异常。

以下是原因：

- 大多数开发人员都熟悉异常，并且知道如何处理它们
- 库不想持有特定立场，使用某一种 **Result 模式** 库或实现自己的版本

但请记住，在构建库时应将异常作为最后可用的选项。通常情况下，返回 null、空集合或布尔 false 值比抛出异常更好。

### 领域实体保护验证

当遵循领域驱动设计原则（DDD）时，你使用构造函数或工厂方法构建领域模型。如果你为领域对象创建传递数据，而这些数据无效（但这种情况本不应该发生）- 你可能会抛出异常。这将表明你的输入验证、映射或其他应用程序层存在需要修复的错误。

## 总结

在.NET 中用 Result 模式替代异常可以使代码更加健壮和可维护。

通过显式处理成功和失败情况，开发人员可以编写更清晰的代码，使其更易于测试和理解。

虽然它可能不适合所有场景，但采用 Result 模式可以显著改善应用程序中的错误处理。

今天就到这里。期待与你再次相见。
