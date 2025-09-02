# [理解 EF Core 中的变更追踪以提升性能](https://antondevtips.com/blog/understanding-change-tracking-for-better-performance-in-ef-core)

Change Tracker 变更跟踪器 是 EF Core 的核心，它负责监控实体的添加、更新和删除。在今天的文章中，你将学习变更跟踪器的工作原理、实体如何被跟踪以及如何将现有实体附加到变更跟踪器。你将获得关于如何通过跟踪技术提高应用程序性能的指导。

最后，我们将探讨 EF Core 变更跟踪器如何在实际场景中显著改进我们的代码。

## EF Core 中的变更跟踪器是什么

变更跟踪器是 EF Core 的一个关键部分，负责跟踪实体实例及其状态。它监控这些实例的变化，并确保数据库相应地更新。这种跟踪机制对于 EF Core 了解哪些实体必须在数据库中插入、更新或删除至关重要。

当你查询数据库时，EF Core 会自动开始跟踪返回的实体。

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var users = await dbContext.Users.ToListAsync();
}
```

从数据库查询用户后，所有实体都会自动添加到变更跟踪器中。更新用户时，变更跟踪器会将 users 集合与其内部从数据库中检索到的 User 实体集合进行比较。EF Core 将使用比较结果来决定生成哪些 SQL 命令来更新数据库中的实体。

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var users = await dbContext.Users.ToListAsync();
    users[0].Email = "test@mail.com";

    await dbContext.SaveChangesAsync();
}
```

在此示例中，我们正在更新第一个用户的电子邮件。在调用 dbContext.SaveChangesAsync() 后，EF Core 会将 users 集合与更改跟踪器中保存的集合进行比较。比较后，EF Core 发现 users 集合已更新，并且更新 SQL 查询将发送到数据库：

```bash
Executed DbCommand (0ms) [Parameters=[@p1='****', @p0='test@mail.com' (Nullable = false) (Size = 13)], CommandType='Text', CommandTimeout='30']
      UPDATE "users" SET "email" = @p0
      WHERE "id" = @p1
      RETURNING 1;
```

要添加和删除实体，应调用 Add 和 Remove 方法：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var users = await dbContext.Users.ToListAsync();
    
    dbContext.Users.Remove(users[1]);
    
    dbContext.Users.Add(new User
    {
        Id = Guid.NewGuid(),
        Email = "one@mail.com"
    });

    await dbContext.SaveChangesAsync();
}
```

变更追踪器将检测到第二个用户被删除，并添加了一个新用户。因此，以下 SQL 命令将被发送到数据库以删除和创建用户：

```bash
Executed DbCommand (0ms) [Parameters=[@p0='***'], CommandType='Text', CommandTimeout='30']
      DELETE FROM "users"
      WHERE "id" = @p0
      RETURNING 1;
      
Executed DbCommand (0ms) [Parameters=[@p0='***', @p1='one@mail.com' (Nullable = false) (Size = 12)], CommandType='Text', CommandTimeout='30']
      INSERT INTO "users" ("id", "email")
      VALUES (@p0, @p1);
```

## 变更追踪器与子实体

EF Core 中的变更追踪器还会追踪与其他实体一起加载的子实体。让我们来探讨以下实体：

```csharp
public class Book
{
    public required Guid Id { get; set; }
    
    public required string Title { get; set; }
    
    public required int Year { get; set; }
    
    public Guid AuthorId { get; set; }

    public Author Author { get; set; } = null!;
}

public class Author
{
    public required Guid Id { get; set; }
    
    public required string Name { get; set; }

    public List<Book> Books { get; set; } = [];
}
```

一个 Book 被一对多映射到 Author 。

当执行以下代码并更新第一本书的作者姓名时：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var books = await dbContext.Books
        .Include(x => x.Author)
        .ToListAsync();
    
    books[0].Author.Name = "Jack Sparrow";

    await dbContext.SaveChangesAsync();
}
```

EF Core 会向数据库生成一个更新请求：

```bash
Executed DbCommand (0ms) [Parameters=[@p1='***', @p0='Jack Sparrow' (Nullable = false) (Size = 12)], CommandType='Text', CommandTimeout='30']
      UPDATE "authors" SET "name" = @p0
      WHERE "id" = @p1
      RETURNING 1;
```

现在我们尝试为第一位作者添加一本新书：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var authors = await dbContext.Authors
        .Include(x => x.Books)
        .ToListAsync();
    
    var newBook = new Book
    {
        Id = Guid.NewGuid(),
        Title = "Asp.Net Core In Action",
        Year = 2024
    };

    authors[0].Books.Add(newBook);
    
    dbContext.Entry(newBook).State = EntityState.Added;

    await dbContext.SaveChangesAsync();
}
```

在这种情况下，你需要手动通知变更追踪器，图书已添加到作者：

```csharp
dbContext.Entry(newBook).State = EntityState.Added;
```

因此，一个外键指向 Author 的插入查询将被发送到数据库：

```bash
Executed DbCommand (11ms) [Parameters=[@p0='fba984cd-a7b8-4eee-998b-165db95068a5', @p1='1072efd7-a71f-40a5-a939-5e68b7e34e0c', @p2='Asp.Net Core In Action' (Nullable = false) (Size = 22), @p3='2024'], CommandType='Text', CommandTimeout='30']
      INSERT INTO "books" ("id", "author_id", "title", "year")
      VALUES (@p0, @p1, @p2, @p3);
```

## EF Core 如何跟踪实体

EF Core 中的实体是根据其状态进行跟踪的，状态可以是以下之一：

- **已添加** - 实体是新的，将被插入到数据库中。
- **已修改** - 实体已被修改，将在数据库中更新。
- **已删除** - 实体已被标记为删除
- **分离** - 实体不应被跟踪，并将从更改跟踪器中移除
- **未更改** - 实体自加载以来未被修改

你可以使用 DbContext 的 Entry 属性检查实体的状态：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var book = dbContext.Books.First();
    var entry = dbContext.Entry(book);
    var state = entry.State; // EntityState.Unchanged
}
```

## 将现有实体附加到更改跟踪器

正如你所见，有时你可能需要将一个现有实体附加到变更追踪器。这在实体从不同上下文或数据库外部（例如，从 API）检索的场景中很常见。

要附加实体，可以使用 Attach 方法，这样更改跟踪器将开始跟踪此实体。此方法默认将实体标记为 Unchanged 。

你需要指定此实体在数据库中应被修改还是删除：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var book = new Book
    {
        Id = Guid.NewGuid(),
        Title = "Asp.Net Core In Action",
        Year = 2024
    };
    
    dbContext.Books.Attach(book);
    dbContext.Entry(book).State = EntityState.Modified;
    
    dbContext.Books.Attach(book);
    dbContext.Entry(book).State = EntityState.Deleted;
}
```

## EF Core 中的批量跟踪操作

EF Core 提供范围操作，用于对多个实体执行批处理操作。这些方法可以简化代码并提高性能。

### AddRange

向上下文添加新的实体集合：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var author = new Author
    {
        Id = Guid.NewGuid(),
        Name = "Andrew Lock"
    };
    
    var books = new List<Book>
    {
        new()
        {
            Id = Guid.NewGuid(),
            Title = "Asp.Net Core In Action 2.0",
            Year = 2020,
            Author = author
        },
        new()
        {
            Id = Guid.NewGuid(),
            Title = "Asp.Net Core In Action 3.0",
            Year = 2024,
            Author = author
        }
    };
    
    dbContext.Books.AddRange(books);
    await dbContext.SaveChangesAsync();
}
```

### UpdateRange

更新上下文中实体集合：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var booksToUpdate = await dbContext.Books
        .Where(x => x.Year >= 2020)
        .ToListAsync();
    
    booksToUpdate.ForEach(b => b.Title += "-updated");
    
    dbContext.Books.UpdateRange(booksToUpdate);
    await dbContext.SaveChangesAsync();
}
```

### RemoveRange

从上下文中移除实体集合：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var blogsToDelete = await dbContext.Books
        .Where(x => x.Year < 2020)
        .ToListAsync();
    
    dbContext.Books.RemoveRange(blogsToDelete);
    await dbContext.SaveChangesAsync();
}
```

### AttachRange

将现有实体集合附加到上下文：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var books = new List<Book>
    {
        // ...
    };
    
    dbContext.Books.AttachRange(books);
    
    foreach (var book in books)
    {
        dbContext.Entry(book).State = EntityState.Modified;
    }
}
```

## 如何禁用变更追踪器

当你从数据库中读取实体，并且不需要更新它们时，你可以通知 EF Core 不在变更跟踪器中跟踪这些实体。当你从数据库中检索大量记录，并且不希望浪费内存来跟踪这些实体（因为它们不会被修改）时，这尤其有用。

**AsNoTracking()** 方法用于查询实体而不对其进行跟踪。这可以提高只读操作的性能，因为 EF Core 省略了跟踪更改的开销：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var books = await dbContext.Books
        .Include(x => x.Author)
        .AsNoTracking()
        .ToListAsync();
}
```

这是优化 EF Core 中只读查询的一个小性能技巧，你需要了解它。

## 如何在 EF Core 中访问跟踪实体

EF Core 允许你访问和操作当前 DbContext 的 Change Tracker 中的跟踪实体。你可以使用 **ChangeTracker.Entries()** 方法检索所有跟踪实体：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var books = await dbContext.Books
        .Include(x => x.Author)
        .ToListAsync();
    
    var trackedEntities = dbContext.ChangeTracker.Entries();
    foreach (var entry in trackedEntities)
    {
        Console.WriteLine($"Entity: {entry.Entity}, State: {entry.State}");
    }
}
```

你还可以根据实体的状态进行筛选：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var books = await dbContext.Books
        .Include(x => x.Author)
        .ToListAsync();
    
    books[0].Author.Name = "Jack Sparrow";
    
    var modifiedEntities = dbContext.ChangeTracker.Entries()
        .Where(e => e.State == EntityState.Modified);
    
    foreach (var entry in modifiedEntities)
    {
        Console.WriteLine($"Modified Entity: {entry.Entity}");
    }
}
```

## 变更跟踪器在实际应用中的一个例子

让我们通过一个真实世界的例子来探讨使用变更追踪器如何显著简化我们的代码。假设你的实体具有 **CreatedAtUtc** 和 **UpdatedAtUtc** 属性。这些属性用于时间审计。

当新实体添加到数据库时， **CreatedAtUtc** 应被赋予当前的 UTC 时间。

每当数据库中现有实体更新时， **UpdatedAtUtc** 应被赋予当前的 UTC 时间。

让我们探讨 User 实体的最基本实现：

```csharp
public class User
{
    public Guid Id { get; set; }
    
    public required string Email { get; set; }
    
    public DateTime CreatedAtUtc { get; set; }
    
    public DateTime? UpdatedAtUtc { get; set; }
}
```

当创建新用户或更新现有用户时，你需要手动指定这些值：

```csharp
using (var dbContext = new ApplicationDbContext())
{
    var user = new User
    {
        Id = Guid.NewGuid(),
        Email = "test@mail.com",
        CreatedAtUtc = DateTime.UtcNow
    };

    dbContext.Users.Add(user);
    await dbContext.SaveChangesAsync();

    user.Email = "another@mail.com";
    user.UpdatedAtUtc = DateTime.UtcNow;

    await dbContext.SaveChangesAsync();
}
```

这看起来没什么大不了，但想象一下你有一个更复杂的应用程序，不仅可以更新用户的电子邮件，还可以更新他的密码、个人数据和权限。而且你可能有很多实体应该具有 **CreatedAtUtc** 和 **UpdatedAtUtc** 属性。

使用手动方法会使你的代码变得混乱，到处都会有代码重复。此外，你可能会忘记设置这些属性，从而在代码中引入错误。

如果我告诉你，你可以使用 EF Core 中的变更追踪器，在一个地方自动为所有需要时间审计的实体设置这些属性，你会怎么想？

首先，让我们引入一个接口：

```csharp
public interface ITimeAuditableEntity
{
    DateTime CreatedAtUtc { get; set; }
    
    DateTime? UpdatedAtUtc { get; set; }
}
```

所有需要时间审计的实体都应该继承这个接口：

```csharp
public class Book : ITimeAuditableEntity
{
    // 其他字段
    
    public DateTime CreatedAtUtc { get; set; }
    
    public DateTime? UpdatedAtUtc { get; set; }
}

public class Author : ITimeAuditableEntity
{
    // 其他字段
    
    public DateTime CreatedAtUtc { get; set; }
    
    public DateTime? UpdatedAtUtc { get; set; }
}
```

现在，在 DbContext 中，你可以重写 SaveChangesAsync 方法来自动设置 CreatedAtUtc 和 UpdatedAtUtc 属性：

```csharp
public class ApplicationDbContext : DbContext
{
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = new CancellationToken())
    {
        var entries = ChangeTracker.Entries<ITimeAuditableEntity>();
    
        foreach (var entry in entries)
        {
            if (entry.State is EntityState.Added)
            {
                entry.Entity.CreatedAtUtc = DateTime.UtcNow;
            }
            else if (entry.State is EntityState.Modified)
            {
                entry.Entity.UpdatedAtUtc = DateTime.UtcNow;
            }
        }
    
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

通过使用 **ChangeTracker.Entries\<ITimeAuditableEntity\>();** ，你可以接收经过筛选的被跟踪实体。之后，为已添加和已更新的实体设置 **CreatedAtUtc** 和 **UpdatedAtUtc** 属性。最后，我们调用 base.SaveChangesAsync 方法将更改保存到数据库。

如果你的应用程序中有多个 DbContext，你可以使用 EF Core 拦截器来实现相同的目标。这样，你就不必在所有 DbContext 中重复代码。

以下是创建此类拦截器的方法：

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

public class TimeAuditableInterceptor : SaveChangesInterceptor
{
    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context!;
        var entries = context.ChangeTracker.Entries<ITimeAuditableEntity>();

        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAtUtc = DateTime.UtcNow;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAtUtc = DateTime.UtcNow;
            }
        }

        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

并在 DbContext 中注册拦截器：

```csharp
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
{
    options.EnableSensitiveDataLogging().UseSqlite(connectionString);
    options.AddInterceptors(new TimeAuditableInterceptor());
});
```

你可以为多个 DbContext 注册此拦截器，并重用单一代码库，对任意数量的实体执行时间审计。

今天就到这里。期待与你再次相见。
