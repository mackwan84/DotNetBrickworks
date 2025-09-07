# [在 EF Core 中使用 BatchUpdate 和 BatchDelete 方法的正确方式](https://antondevtips.com/blog/correct-way-to-use-batch-update-and-batch-delete-methods-in-ef-core)

在你的 EF Core 应用程序中执行批量更新或删除时是否遇到过性能问题？

EF Core 使用 ExecuteUpdate 和 ExecuteDelete 方法提供高效的批量操作，显著提升性能。这些操作允许在单个 SQL 查询中更新和删除多个实体，而无需从数据库中检索它们。

今天，我将向你展示如何在 EF Core 中正确使用 ExecuteUpdate 和 ExecuteDelete 方法，以确保数据一致性。

## 更新和删除实体的默认方法

首先，让我们探讨 EF Core 中实体的更新和删除是如何工作的

默认方法涉及将实体加载到 EF Core 变更跟踪器中，该跟踪器会将它们保存在内存中。

这种跟踪机制对于 EF Core 了解必须在数据库中插入、更新或删除哪些实体至关重要。

```csharp
var users = await dbContext.Users.ToListAsync();
```

从数据库查询用户后，所有实体都会自动添加到变更跟踪器中。当更新用户时 - EF Core 会将当前用户集合与存储在变更跟踪器中的保存集合进行比较。EF Core 将使用比较结果来决定生成什么样的 SQL 命令来更新数据库中的实体：

```bash
Executed DbCommand (0ms) [Parameters=[@p1='****', @p0='test@mail.com' (Nullable = false) (Size = 13)], CommandType='Text', CommandTimeout='30']
      UPDATE "users" SET "email" = @p0
      WHERE "id" = @p1
      RETURNING 1;
```

让我们探索一个更新给定 Author 的书籍价格的示例：

```csharp
public sealed record UpdateBooksPriceRequest(decimal Delta);

app.MapPut("/authors/{authorId:guid}/books/update-price",
    async (Guid authorId,
    UpdateBooksPriceRequest request,
    ApplicationDbContext dbContext) =>
{
    var books = await dbContext.Books.Where(b => b.AuthorId == authorId).ToListAsync();
    foreach (var book in books)
    {
        book.Price += request.Delta;
        book.UpdatedAtUtc = DateTime.UtcNow;
    }

    await dbContext.SaveChangesAsync();
    return Results.Ok(new { updated = books.Count });
});
```

这种方法非常直接：从数据库加载实体，更新所需属性，然后 EF Core 会确定要生成哪些 SQL 语句来更新实体：

```bash
UPDATE devtips_batch_operations.books SET price = @p0, updated_at_utc = @p1
      WHERE id = @p2;
      UPDATE devtips_batch_operations.books SET price = @p3, updated_at_utc = @p4
      WHERE id = @p5;
      UPDATE devtips_batch_operations.books SET price = @p6, updated_at_utc = @p7
      WHERE id = @p8;
```

让我们探索另一个删除给定作者的多本书籍的示例：

```csharp
app.MapDelete("/authors/{authorId:guid}/books",
    async (Guid authorId, ApplicationDbContext dbContext) =>
{
    var booksToDelete = await dbContext.Books
        .Where(b => b.AuthorId == authorId)
        .ToListAsync();

    if (booksToDelete.Count == 0)
    {
        return Results.NotFound("No books found for the given author.");
    }

    dbContext.Books.RemoveRange(booksToDelete);
    await dbContext.SaveChangesAsync();

    return Results.Ok(new { deletedCount = booksToDelete.Count });
});
```

这种方法非常直接：从数据库加载实体，调用 RemoveRange 方法，EF Core 将确定要生成哪些 SQL 语句来删除实体：

```bash
DELETE FROM devtips_batch_operations.books
  WHERE id = @p0;
  DELETE FROM devtips_batch_operations.books
  WHERE id = @p1;
  DELETE FROM devtips_batch_operations.books
  WHERE id = @p2;
```

如你所见，这两种操作都会为每个更新和删除的实体生成单独的 SQL 命令，这可能会导致效率低下。虽然对于小型数据集来说简单有效，但对于中等和大量记录而言，这种方法可能会效率低下。

让我们探索一个更高效的解决方案。

## 使用 ExecuteUpdate 和 ExecuteDelete 方法

EF Core 7 引入了 ExecuteUpdate 和 ExecuteDelete 方法用于批量操作。这些方法绕过变更跟踪器，允许通过单个 SQL 语句直接在数据库中执行更新和删除操作。

这些方法具有以下优点：

- 消除将实体从数据库加载到 ChangeTracker 的开销
- 更新和删除操作作为单个 SQL 命令执行，使此类查询非常高效

让我们探索如何使用这些方法重写之前的示例。

这就是你可以用 ExecuteUpdate 更新书籍价格的方法：

```csharp
app.MapPut("/authors/{authorId:guid}/books/batch-update-price",
    async (Guid authorId,
        UpdateBooksPriceRequest request,
        ApplicationDbContext dbContext) =>
{
    var updatedCount = await dbContext.Books
        .Where(b => b.AuthorId == authorId)
        .ExecuteUpdateAsync(s => s
            .SetProperty(b => b.Price, u => u.Price + request.Delta)
            .SetProperty(b => b.UpdatedAtUtc, DateTime.UtcNow));

    return Results.Ok(new { updated = updatedCount });
});
```

首先我们根据给定的 Author 标识符筛选书籍，然后通过调用 SetProperty 方法更新所需的属性。

这会生成单个 SQL 命令：

```bash
UPDATE devtips_batch_operations.books AS b
SET updated_at_utc = now(),
  price = b.price + @__request_Delta_1
WHERE b.author_id = @__authorId_0
```

让我们来探索一个删除图书的示例：

```csharp
app.MapDelete("/authors/{authorId:guid}/books/batch",
    async (Guid authorId, ApplicationDbContext context) =>
{
    var deletedCount = await dbContext.Books
        .Where(b => b.AuthorId == authorId)
        .ExecuteDeleteAsync();

    return Results.Ok(new { deleted = deletedCount });
});
```

这也会生成单个 SQL 命令：

```bash
DELETE FROM devtips_batch_operations.books AS b
  WHERE b.author_id = @__authorId_0
```

对于较大的修改，这些方法的效率要高得多。

即使更新或删除单个实体，这些方法也很有益。你执行的是单个 SQL 命令，而不是两个独立的操作（先加载再更新或删除）。如果你有多个实体，则需要向数据库发送 1+N 次请求。

这会显著降低你的应用程序速度。

但请记住， ExecuteUpdate 和 ExecuteDelete 方法有一个主要注意事项。它们与 EF Core 的变更跟踪器分离。如果之后调用 SaveChanges 并且失败，通过 ExecuteUpdate 和 ExecuteDelete 所做的更改将不会被回滚。

让我们来探讨如何解决这个问题！

## 如何使用 ExecuteUpdate 和 ExecuteDelete 方法确保数据一致性

在执行多个批量操作时，或将批量操作与 SaveChanges 一起执行时，你需要确保数据的一致性。

你需要手动将所有数据库命令包装在一个事务中。让我们来探索一个示例：

```csharp
app.MapPut("/authors/{authorId:guid}/books/multi-update", 
    async(Guid authorId,
        UpdateBooksPriceRequest request,
        ApplicationDbContext dbContext) =>
{
    await using var transaction = await dbContext.Database.BeginTransactionAsync();

    try
    {
        var authorBooks = await dbContext.Books
            .Where(b => b.AuthorId == authorId)
            .Select(x => new  { x.Id, x.Price })
            .ToListAsync();

        var updatedCount = await dbContext.Books
            .Where(b => b.AuthorId == authorId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(b => b.Price, u => u.Price + request.Delta)
                .SetProperty(b => b.UpdatedAtUtc, DateTime.UtcNow));

        await dbContext.Authors
            .Where(b => b.Id == authorId)
            .ExecuteUpdateAsync(s => s
                .SetProperty(b => b.UpdatedAtUtc, DateTime.UtcNow));

        var priceRecords = authorBooks.Select(x => new PriceRecord
        {
            Id = Guid.NewGuid(),
            BookId = x.Id,
            OldPrice = x.Price,
            NewPrice = x.Price + request.Delta,
            CreatedAtUtc = DateTime.UtcNow
        }).ToList();

        dbContext.PriceRecords.AddRange(priceRecords);

        await dbContext.SaveChangesAsync();
        await transaction.CommitAsync();

        return Results.Ok(new { updated = updatedCount });
    }
    catch (Exception)
    {
        await transaction.RollbackAsync();
        return Results.BadRequest("Error updating books");
    }
});
```

在这个 API 端点中，有 3 个更新操作：

- 更新 Book 价格
- 正在更新 Author 行的时间戳
- 创建价格变更记录

将这些操作包装在事务中可确保要么所有操作都成功，要么都不执行，从而维护数据库完整性。

## 总结

ExecuteUpdate 和 ExecuteDelete 方法显著提升了 EF Core 批量操作的性能。然而，为避免数据一致性问题，如果将这些方法与其他操作结合使用，请务必将它们包装在手动事务中。这确保了稳健、快速且一致的数据库状态管理。

今天就到这里。期待与你再次相见。
