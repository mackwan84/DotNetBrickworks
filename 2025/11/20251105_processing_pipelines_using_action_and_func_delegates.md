# [使用 Action 和 Func 委托处理工作流](https://medium.com/@devesh.akgec/processing-pipelines-using-action-and-func-delegates-dcba212b0516)

我们将使用 Action 和 Func 委托设计可扩展的审批工作流。

## 背景

我们将设计如下审批系统

我们有一个需要经过多个审批阶段的请求：

- 经理审批
- 财务审批
- 总监审批

在每个阶段，我们：检查是否满足审批标准。可能会拒绝或批准该请求

## 传统方法

假设存在以下请求

```csharp
public class Request
{
    public string RequestId { get; set; } = Guid.NewGuid().ToString();
    public decimal Amount { get; set; }
    public string Status { get; set; } = "Pending";
}
```

审批系统

```csharp
public class ApprovalSystem
 {
    public Request Process(Request request)
    {
        if (!ManagerApproval(request))
           return request;

        if (!FinanceApproval(request))
           return request;

        if (!DirectorApproval(request))
           return request;

        request.Status = "Approved";
        return request;
    }

    private bool ManagerApproval(Request request)
    {
        if (request.Amount <= 1000)
        {
           Console.WriteLine("Manager approved.");
           return true;
        }
        Console.WriteLine("Manager rejected (amount too high).");
        request.Status = "Rejected by Manager";
        return false;
    }

    private bool FinanceApproval(Request request)
    {
        if (request.Amount <= 5000)
        {
           Console.WriteLine("Finance approved.");
           return true;
        }
        Console.WriteLine("Finance rejected (amount too high).");
        request.Status = "Rejected by Finance";
        return false;
    }

    private bool DirectorApproval(Request request)
    {
        if (request.Amount <= 10000)
        {
           Console.WriteLine("Director approved.");
           return true;
        }
        Console.WriteLine("Director rejected (amount too high).");
        request.Status = "Rejected by Director";
        return false;
    }
 }
 ```

我们创建了多个函数，每次有新需求时都需要更新下面的函数

```csharp
public Request Process(Request request)
{
    if (!ManagerApproval(request))
       return request;

    if (!FinanceApproval(request))
       return request;

    if (!DirectorApproval(request))
       return request;

    request.Status = "Approved";
    return request;
}
```

这段代码可能会遇到很多问题，比如单一职责原则的问题，此外代码不具备可扩展性，每次都必须更新处理函数。

## 使用 Action 和 Func 委托的改进方法

让我们使用 Action 和 Func 将上述函数转换如下

```csharp
public static class ApprovalStepsDefinations
{
    public static Func<Request, Request> ManagerApproval = req =>
    {
        if (req.Amount <= 1000)
        {
           Console.WriteLine(" Manager approved.");
        }
        else
        {
           req.Status = "Rejected by Manager";
           Console.WriteLine(" Manager rejected.");
        }
        return req;
    };

    public static Func<Request, Request> FinanceApproval = req =>
    {
        if (req.Amount <= 5000)
        {
           Console.WriteLine("Finance approved.");
        }
        else
        {
           req.Status = "Rejected by Finance";
           Console.WriteLine(" Finance rejected.");
        }
        return req;
    };

    public static Func<Request, Request> DirectorApproval = req =>
    {
        if (req.Amount <= 10000)
        {
           req.Status = "Approved";
           Console.WriteLine(" Director approved.");
        }
        else
        {
           req.Status = "Rejected by Director";
           Console.WriteLine( "Director rejected.");
        }
        return req;
    };
}
```

我们使用 Func 创建了多个函数，这些函数接收 Object 类型的 Request 并返回相同的类型

## 集中化管道执行器

```csharp
public class ApprovalPipelineExecutor
{
    private readonly List<Func<Request, Request>> _steps = new();
    private readonly List<Action<Request>> _actions = new();

    public ApprovalPipelineExecutor AddStep(Func<Request, Request> step)
    {
        _steps.Add(step);
        return this;
    }

    public ApprovalPipelineExecutor AddAction(Action<Request> action)
    {
        _actions.Add(action);
        return this;
    }

    public Request Execute(Request request)
    {
        var result = request;

        foreach (var step in _steps)
        {
            result = step(result);

            foreach (var action in _actions)
                action(result);

            // 如果请求被拒绝，停止处理
            if (result.Status.StartsWith("Rejected"))
                break;
        }

        return result;
    }
}
```

这个类可以作为泛型类使用，我们在这里定义了要执行的步骤。Execute 函数使用 for each 循环执行每个函数

## 使用示例

```csharp
var approvalPipeline = new ApprovalPipelineExecutor()
    .AddStep(ApprovalStepsDefinations.ManagerApproval)
    .AddStep(ApprovalStepsDefinations.FinanceApproval)
    .AddStep(ApprovalStepsDefinations.DirectorApproval)
    .AddAction(req => Console.WriteLine($"Status Update: {req.Status}"));

var request = new Request { Amount = 3000m };

var result = approvalPipeline.Execute(request);

Console.WriteLine($"Final Status: {result.Status}");
```

我们能够一个接一个地添加步骤，是因为使用了"return this"

```csharp
public ApprovalPipelineExecutor AddStep(Func<Request, Request> step)
{
    _steps.Add(step);
    return this;
}
```

这段代码非常直观易懂，添加和删除步骤也非常简单。

这段代码具有巨大的可扩展性和可测试性。每当有新步骤出现时，我们只需要添加它，而集中执行器类将保持不变，我们不需要在多个地方进行修改。

![输出结果](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*yXirSY6F_LV7XHxz9BSzEA.png)

今天就到这里。希望这些内容对你有帮助。
