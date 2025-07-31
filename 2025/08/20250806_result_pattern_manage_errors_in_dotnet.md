# [.NET 中使用 Result 模式管理错误](https://www.nikolatech.net/blogs/result-pattern-manage-errors-in-dotnet)

**Result 模式** 正变得越来越流行，特别是在.NET 生态系统中。

它受函数式编程影响，代表了传统异常处理的一种有趣替代方案。其思想是通过明确表示操作结果（如成功或失败）来增强流程控制，而不依赖于异常。

此模式将结果数据和状态封装在单个对象中，使开发人员能够轻松检查结果并决定如何继续。

在 .NET 中，没有内置的 Result 类型。但是，你可以轻松实现自己的自定义 Result 类型，或使用提供此功能的第三方库。

而今天，我们将介绍 **Result 模式** 的自定义和第三方实现方法。

废话不多说，让我们开始实现。

## 手动实现

虽然有很棒的第三方库可用，但手动实现非常直接，并且能让你有更多控制权来定制解决方案以满足特定需求。

此外，由于它易于实现，在项目中最小化依赖通常是一个实用的选择。

首先，你需要定义 Result 类型。重要的是要记住，没有单一的解决方案，不同的实现方式仍然可以有效地帮助你实现目标。

例如，你可以设计一个通用的 Result 类型，该类型包含成功路径和失败路径的泛型，使其能够接受各种错误类型。

或者，你可以选择一种更简单的方法，即接受单一错误类型，并仅将泛型属性用于成功结果。

以下是一个 Result 类型的简单示例：

```csharp
public sealed class Result<T>
{
    public T? Value { get; }

    public Error? Error { get; }

    public bool IsSuccess { get; }

    public bool IsError => !IsSuccess;

    private Result(T value)
    {
        Value = value ?? throw new ArgumentNullException(nameof(value), "Value cannot be null.");
        IsSuccess = true;
        Error = null;
    }

    private Result(Error error)
    {
        Error = error ?? throw new ArgumentNullException(nameof(error), "Error cannot be null.");
        IsSuccess = false;
        Error = error;
    }

    public static Result<T> Success(T value) => new(value);

    public static Result<T> Failure(Error error) => new(error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<Error, TResult> onError)
    {
        return IsSuccess ? onSuccess(Value!) : onError(Error!);
    }
}
```

我的 Result 类可以存储类型为 **T** 的成功值或一个 **Error**，并通过 **IsSuccess** 和 **IsError** 属性来指示结果的状态。

**Success()** 和 **Failure()** 方法用于根据结果是成功还是失败来返回一个 Result。

**Match()** 方法通过为成功和错误情况提供不同的处理方式来实现分支逻辑，允许调用者单独管理每种结果。

**Error** 通过为不同错误类型提供结构化数据，标准化了错误处理。这种方法确保了在整个应用程序中保持一致的错误，使其易于理解和使用。

```csharp
public sealed record Error(int Code, string Description)
{
    public static Error ProductNotFound => new(100, "Product not found");

    public static Error ProductBadRequest => new(101, "Product bad request");
}
```

此外，我创建了一个 Unit 结构体来表示 void 返回类型，用于我们想要指示成功但不返回值的情况。

```csharp
public struct Unit
{
    public static readonly Unit Value = new Unit();
}
```

下面是**Result 模式**的一个实际示例：

```csharp
public async Task<Result<Guid>> Handle(Command request, CancellationToken cancellationToken)
{
    var validationResult = await _validator.ValidateAsync(request, cancellationToken);
    if (!validationResult.IsValid)
    {
        return Result<Guid>.Failure(Error.ProductBadRequest);
    }

    var product = new Product(
        Guid.NewGuid(),
        DateTime.UtcNow,
        request.Name,
        request.Description,
        request.Price);

    _dbContext.Products.Add(product);

    await _dbContext.SaveChangesAsync(cancellationToken);

    return Result<Guid>.Success(product.Id);
}
```

```csharp
app.MapPost("products/", async (
    ISender sender,
    CreateRequest request,
    CancellationToken cancellationToken) =>
{
    var command = request.Adapt<Command>();

    var response = await sender.Send(command, cancellationToken);

    return response.Match(
        x => Results.Ok(x),
        error => Results.BadRequest(error)
    );
});
```

## 第三方库

.NET 提供了多种优秀的第三方库来实现**Result 模式**：

- ardalis/Result
- amantinband/Error-Or
- vkhorikov/CSharpFunctionalExtensions
- YoussefSell/Result.Net
- altmann/FluentResults

不幸的是，我没有机会尝试所有这些，但以下是我对 ErrorOr、FluentResult 和 CSharpFunctionalExtensions 的看法。

**ErrorOr** 和 **FluentResult** 都是从零开始项目时的绝佳选择。它们支持多个错误而不仅仅是一个，提供各种扩展方法，提供流畅接口等等。

但是，如果你已经有一个包含许多异常的项目，并希望转向**Result 模式**，该怎么办呢？

这就是 **CSharpFunctionalExtensions** 的优势所在，它提供了尽可能简单的转换方式。

**CSharpFunctionalExtensions** 是 C#中一个流行的库，旨在将函数式编程概念引入该语言。

Result 类型是其中之一，但这个库的独特之处在于它能够自动包装异常。你无需重构整个代码库来移除异常，只需使用 Result.Try()方法将它们包装在 Result 中即可。

这是一个简单的示例：

```csharp
await _validator.ValidateAndThrowAsync(request, cancellationToken);
```

你可以这样包装异常，而不是处理它们：

```csharp
var result = await Result.Try(async () => await _validator.ValidateAndThrowAsync(request, cancellationToken));
if (result.IsFailure)
{
    return result;
}
```

此方法执行可能会抛出异常的代码，如果发生异常，它会捕获该异常并返回一个包含异常消息的失败结果。

使用其他库也可以实现类似的结果，但这个库非常适合我的用例。

## 结论

**Result 模式**并非万能解决方案，异常本身也并非天生就有问题。

它是一个有价值的工具，特别是在需要清晰错误处理的场景中。

Result 类型通常包含有关错误的详细上下文，例如消息或特定错误类型。除此之外，异常确实会引入额外的开销。

然而，即使你选择仅使用**Result 模式**，你的代码仍需要处理异常，因为你可能使用的框架和许多库会抛出异常。

**Result 模式**需要实现条件逻辑，在处理多个潜在错误时，这种逻辑可能会变得越来越复杂。当发生错误时，必须将其传播到顶层，需要在每一层进行验证。

此外，如果你能够实现中间件，就可以集中处理错误并保持清晰的关注点分离，将业务逻辑与错误处理逻辑区分开来。

关键在于仔细考虑是否以及如何实现**Result 模式**。

今天就到这里。期待与你再次相见。
