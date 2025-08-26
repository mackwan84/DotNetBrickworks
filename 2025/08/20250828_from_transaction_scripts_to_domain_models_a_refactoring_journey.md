# [从事务脚本到领域模型：记一次重构之旅](https://www.milanjovanovic.tech/blog/from-transaction-scripts-to-domain-models-a-refactoring-journey)

我曾经领导开发了一款健身追踪应用。我们最初使用事务脚本来处理锻炼创建和运动记录等功能。对于应用的早期阶段，这是一种简单而有效的方法。

然而，随着我们添加更复杂的功能，我们的业务逻辑变得臃肿。新的业务规则与现有逻辑交织在一起，使代码难以维护。每次更改都可能带来意想不到的后果。

我们通过引入领域模型解决了问题。领域模型将焦点从过程转向领域对象。我们的代码变得更加富有表现力，更容易理解，也更不容易出错。

这次经历给了我一个宝贵的教训，我想在这里与大家分享一下。

## 事务脚本

从本质上讲，商业应用程序通过与用户进行独立的交互（交易）来运行。这些交易可能从简单的数据检索到涉及多项验证、计算以及对系统数据库进行更新的复杂操作。

事务脚本模式提供了一种简单的方法来封装每个事务背后的逻辑。它将所有必要的步骤，从数据访问到业务规则，组织到一个单一的、自包含的过程中。

> 通过过程来组织业务逻辑，每个过程处理来自表现层的单个请求。
> 事务脚本，《企业应用架构模式》

以下是将锻炼添加到训练计划的示例：

```csharp
internal sealed class AddExercisesCommandHandler(
    IWorkoutRepository workoutRepository,
    IUnitOfWork unitOfWork)
    : ICommandHandler<AddExercisesCommand>
{
    public async Task<Result> Handle(
        AddExercisesCommand request,
        CancellationToken cancellationToken)
    {
        Workout? workout = await workoutRepository.GetByIdAsync(
            request.WorkoutId,
            cancellationToken);

        if (workout is null)
        {
            return Result.Failure(WorkoutErrors.NotFound(request.WorkoutId));
        }

        List<Error> errors = [];
        foreach (ExerciseRequest exerciseDto in request.Exercises)
        {
            if (exerciseDto.TargetType == TargetType.Distance &&
                exerciseDto.DistanceInMeters is null)
            {
                errors.Add(ExerciseErrors.MissingDistance);

                continue;
            }

            if (exerciseDto.TargetType == TargetType.Time &&
                exerciseDto.DurationInSeconds is null)
            {
                errors.Add(ExerciseErrors.MissingDuration);

                continue;
            }

            var exercise = new Exercise(
                Guid.NewGuid(),
                workout.Id,
                exerciseDto.ExerciseType,
                exerciseDto.TargetType,
                exerciseDto.DistanceInMeters,
                exerciseDto.DurationInSeconds);

            workout.Exercises.Add(exercise);
        }

        if (errors.Count != 0)
        {
            return Result.Failure(new ValidationError(errors.ToArray()));
        }

        await unitOfWork.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

这里没有太多逻辑。我们只是检查锻炼是否存在以及练习是否有效。

当我们开始添加更多逻辑时会发生什么？

让我们添加另一条业务规则。我们必须限制单次锻炼中允许的练习数量（例如，不超过 10 个练习）。

```csharp
internal sealed class AddExercisesCommandHandler(
    IWorkoutRepository workoutRepository,
    IUnitOfWork unitOfWork)
    : ICommandHandler<AddExercisesCommand>
{
    public async Task<Result> Handle(
        AddExercisesCommand request,
        CancellationToken cancellationToken)
    {
        Workout? workout = await workoutRepository.GetByIdAsync(
            request.WorkoutId,
            cancellationToken);

        if (workout is null)
        {
            return Result.Failure(WorkoutErrors.NotFound(request.WorkoutId));
        }

        List<Error> errors = [];
        foreach (ExerciseRequest exerciseDto in request.Exercises)
        {
            if (exerciseDto.TargetType == TargetType.Distance &&
                exerciseDto.DistanceInMeters is null)
            {
                errors.Add(ExerciseErrors.MissingDistance);

                continue;
            }

            if (exerciseDto.TargetType == TargetType.Time &&
                exerciseDto.DurationInSeconds is null)
            {
                errors.Add(ExerciseErrors.MissingDuration);

                continue;
            }

            var exercise = new Exercise(
                Guid.NewGuid(),
                workout.Id,
                exerciseDto.ExerciseType,
                exerciseDto.TargetType,
                exerciseDto.DistanceInMeters,
                exerciseDto.DurationInSeconds);

            workout.Exercises.Add(exercise);

            if (workout.Exercise.Count > 10)
            {
                return Result.Failure(
                    WorkoutErrors.MaxExercisesReached(workout.Id));
            }
        }

        if (errors.Count != 0)
        {
            return Result.Failure(new ValidationError(errors.ToArray()));
        }

        await unitOfWork.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

我们可以继续向事务脚本添加更多业务逻辑。例如，我们可以引入运动类型限制或强制执行特定的运动顺序。你可以想象，随着时间的推移，复杂性将如何不断增加。

另一个关注点是事务脚本之间的代码重复。如果我们在多个事务脚本中需要类似的逻辑，这种情况就可能发生。你可能会想通过从一个事务脚本调用另一个事务脚本来解决这个问题，但这会引入一系列不同的问题。

那么，我们该如何解决这些问题呢？

## 重构为领域模型

什么是领域模型？

> 一个既包含行为又包含数据的领域对象模型。
> 领域模型，《企业应用架构模式》

领域模型允许你将领域逻辑（行为）和状态变更（数据）封装在一个对象内部。在领域驱动设计的术语中，我们会将其称为聚合。

在 DDD 中，聚合是一组被视为数据变更单一单元的对象集群。聚合代表了一致性边界。它通过确保某些不变量始终对整个聚合成立来帮助维护一致性。在我们的健身示例中， **Workout** 类可以被视为一个聚合根，它包含了其中的所有练习。

这与事务脚本有什么关系？

我们可以将领域逻辑和状态变更从事务脚本移动到聚合中。这通常被称为将逻辑"下推"到领域。

以下是提取领域逻辑后领域模型将呈现的结构：

```csharp
public sealed class Workout
{
    private readonly List<Exercise> _exercises = [];

    // 此处省略构造函数和其他属性

    public Result AddExercises(ExerciseModel[] exercises)
    {
        List<Error> errors = [];
        foreach (var exerciseModel in exercises)
        {
            if (exerciseModel.TargetType == TargetType.Distance &&
                exerciseModel.DistanceInMeters is null)
            {
                errors.Add(ExerciseErrors.MissingDistance);

                continue;
            }

            if (exerciseModel.TargetType == TargetType.Time &&
                exerciseModel.DurationInSeconds is null)
            {
                errors.Add(ExerciseErrors.MissingDuration);

                continue;
            }

            var exercise = new Exercise(
                Guid.NewGuid(),
                this.Id,
                exerciseDto.ExerciseType,
                exerciseDto.TargetType,
                exerciseDto.DistanceInMeters,
                exerciseDto.DurationInSeconds);

            this.Exercises.Add(exercise);

            if (this.Exercises.Count > 10)
            {
                return Result.Failure(
                    WorkoutErrors.MaxExercisesReached(this.Id));
            }
        }

        if (errors.Count != 0)
        {
            return Result.Failure(new ValidationError(errors.ToArray()));
        }

        return Result.Success();
    }
}
```

随着领域逻辑被置于领域模型内部，我们可以轻松地在事务脚本之间共享这些逻辑。测试领域模型比测试事务脚本更为简单。对于事务脚本，我们必须提供任何依赖项（可能作为模拟对象）来进行测试。然而，我们可以独立地测试领域模型。

更新后的事务脚本变得更加直接明了，并专注于其主要任务：

```csharp
internal sealed class AddExercisesCommandHandler(
    IWorkoutRepository workoutRepository,
    IUnitOfWork unitOfWork)
    : ICommandHandler<AddExercisesCommand>
{
    public async Task<Result> Handle(
        AddExercisesCommand request,
        CancellationToken cancellationToken)
    {
        Workout? workout = await workoutRepository.GetByIdAsync(
            request.WorkoutId,
            cancellationToken);

        if (workout is null)
        {
            return Result.Failure(WorkoutErrors.NotFound(request.WorkoutId));
        }

        var exercises = request.Exercises.Select(e => e.ToModel()).ToArray();

        var result = workout.AddExercises(exercises);

        if (result.IsFailure)
        {
            return result;
        }

        await unitOfWork.SaveChangesAsync(cancellationToken);

        return Result.Success();
    }
}
```

## 总结

事务脚本是简单应用程序的实用起点。它们提供了一种直接的方法来实现用例。事务脚本是构建垂直切片的推荐方法。然而，随着应用程序的增长，事务脚本可能会变得难以维护。

重构为领域模型可以让你将业务逻辑封装在领域对象中。这促进了代码的可重用性，并使你的应用程序更能适应变化。将逻辑下推还能提高可测试性和可维护性。

你应该使用事务脚本还是领域模型？

这是一个你应该考虑的实用方法。从事务脚本开始，但要关注不断增长的复杂性。当你注意到一个事务脚本承担了过多职责时，考虑添加领域模型。记住，领域模型应该封装领域逻辑的部分复杂性。

今天就到这里。期待与你再次相见。
