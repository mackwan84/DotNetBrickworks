# [.NET 中的锁与并发控制简介](https://www.milanjovanovic.tech/blog/introduction-to-locking-and-concurrency-control-in-dotnet-6)

今天我们将了解如何在 .NET 中使用锁功能。

我们不会讨论锁在操作系统层面的实际实现方式。相反，我将专注于应用级的锁机制。

锁允许我们控制有多少个线程可以访问某段代码。为什么要这样做呢？

通常是因为你想保护对昂贵资源的访问，并且需要锁所提供的并发控制。

我们将使用一个简单的 BankAccount 类和一个 Deposit 方法来说明如何实现锁。

## C# Lock 语句

C# 语言支持使用 lock 语句进行锁。你可以使用 lock 语句定义一个只有一个线程可以访问的代码块。

lock 语句获取给定对象的互斥锁（mutex），执行语句块，然后释放锁。

```csharp
lock(_lock)
{
   // 你的代码
}
```

这里 _lock 是一个引用类型，通常是一个 object 实例。

让我们看看如何使用 lock 语句来实现 BankAccount 类：

```csharp
public class BankAccount
{
   private static readonly object _lock = new();
   private decimal _balance;

   public void Deposit(decimal amount)
   {
      lock(_lock)
      {
         _balance += amount;
      }
   }
}
```

第一个到达并执行 lock 语句的线程将被允许更新 _balance 。任何其他线程将被阻塞，直到锁被释放。

## 使用 Semaphore 进行锁

Semaphore 类是我们可以用来实现相同效果的另一个选项。

我们将使用 Semaphore 构造函数将 initialCount 设置为 1，这意味着 Semaphore 在开始时是打开的。我们还会将 maximumCount 设置为 1，这意味着只允许一个线程进入 Semaphore 。

让我们看看如何使用 Semaphore 来实现 BankAccount 类：

```csharp
public class BankAccount
{
   private static readonly Semaphore _semaphore = new(
      initialCount: 1,
      maximumCount: 1);

   private decimal _balance;

   public void Deposit(decimal amount)
   {
      _semaphore.WaitOne();

      _balance += amount;

      _semaphore.Release();
   }
}
```

要进入 Semaphore ，我们必须调用 WaitOne 方法。

如果之前没有线程在里面，我们的线程就被允许进入 Semaphore 并更新余额。

更新余额后，我们调用 Release 方法释放 Semaphore ，以便其他可能正在等待的线程可以使用。

## 使用 SemaphoreSlim 进行异步锁

如果我们想要在锁的上下文中调用异步方法，该怎么办？

我们不能使用 lock 语句，因为它不支持异步调用。在 lock 语句内等待异步调用会导致编译错误。

Semaphore 类可以解决这个问题。

但我想向你展示我们拥有的另一个选项， SemaphoreSlim 。它是 Semaphore 类的轻量级替代方案，并且具有 async 方法。

让我们看看如何使用 SemaphoreSlim 来实现 BankAccount 类：

```csharp
public class BankAccount
{
   private static readonly SemaphoreSlim _semaphore = new(
      initialCount: 1,
      maximumCount: 1);

   private decimal _balance;

   public async Task Deposit(decimal amount)
   {
      await _semaphore.WaitAsync();

      _balance += amount;

      _semaphore.Release();
   }
}
```

请注意，我将 Deposit 方法更新为返回 Task 。

这次，我们调用 WaitAsync 来阻塞当前线程，直到它可以进入信号量。

更新余额后，我们调用 Release 方法来释放 SemaphoreSlim ，就像前面的示例中那样。

## .NET 中还有其他锁选项吗？

到目前为止，我提到了三种实现锁的选项：

- lock 语句
- Semaphore
- SemaphoreSlim

然而，.NET 还有其他用于并发控制的类可供你探索，例如 Monitor 、 Mutex 、 ReaderWriterLock 等等。

今天就到这里。期待与你再次相见。
