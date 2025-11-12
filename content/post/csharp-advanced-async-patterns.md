---
title: "C# Async Patterns: Beyond await - Cancellation, Timeouts, and the Dark Arts"
date: 2025-07-10T09:15:00+01:00
draft: false
categories:
  - .NET
tags:
  - C#
  - .NET
  - Async
  - Performance
  - Advanced
description: "Advanced async/await patterns that most C# developers never learn until they hit production issues."
---

Everyone knows `async/await` in C#. It's the first thing you learn when moving to modern .NET. But after debugging production issues at 2 AM-deadlocks, memory leaks, and operations that never complete-I've learned that there's a huge gap between knowing `async/await` and actually understanding asynchronous programming in C#.

## The Cancellation Token Deep Dive

`CancellationToken` is probably the most underutilized feature of async C#. Most developers pass `CancellationToken.None` everywhere and wonder why their applications hang during shutdown.

### Proper Cancellation Propagation

The rule is simple: if your method is async and takes longer than a few milliseconds, it should accept a `CancellationToken`. And you should pass it to every async operation you call:

```csharp
public async Task ProcessDataAsync(
    IEnumerable<DataItem> items,
    CancellationToken cancellationToken = default)
{
    foreach (var item in items)
    {
        cancellationToken.ThrowIfCancellationRequested(); // Check before expensive operations
        
        await ProcessItemAsync(item, cancellationToken); // Always pass it down
        await SaveToDatabaseAsync(item, cancellationToken);
    }
}
```

But here's the gotcha: `ThrowIfCancellationRequested()` throws `OperationCanceledException`, which you might want to catch and handle differently than other exceptions. Some frameworks (like ASP.NET Core) treat `OperationCanceledException` specially, so be aware of that.

### Cancellation Token Sources and Timeouts

`CancellationTokenSource` is how you create cancellation tokens. The trick is combining multiple cancellation sources:

```csharp
public async Task<string> FetchWithTimeoutAsync(
    string url,
    TimeSpan timeout,
    CancellationToken cancellationToken = default)
{
    using var timeoutCts = new CancellationTokenSource(timeout);
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        cancellationToken,
        timeoutCts.Token);

    try
    {
        using var httpClient = new HttpClient();
        var response = await httpClient.GetAsync(url, linkedCts.Token);
        return await response.Content.ReadAsStringAsync(linkedCts.Token);
    }
    catch (OperationCanceledException) when (timeoutCts.Token.IsCancellationRequested)
    {
        throw new TimeoutException($"Request to {url} timed out after {timeout}");
    }
}
```

The `CreateLinkedTokenSource` method creates a token that cancels when *any* of the source tokens cancel. This is the pattern for implementing timeouts with cancellation support.

### Registering Callbacks

You can register callbacks that run when a cancellation token is canceled:

```csharp
public async Task LongRunningOperationAsync(CancellationToken cancellationToken)
{
    using var registration = cancellationToken.Register(() =>
    {
        // Cleanup code that runs synchronously when canceled
        CleanupResources();
    });

    await DoWorkAsync(cancellationToken);
}
```

But be careful: these callbacks run synchronously on the thread that triggers cancellation. Don't do heavy work here, and definitely don't await anything. If you need async cleanup, you're better off checking the token periodically and handling cleanup in your main async method.

## ConfigureAwait: The Controversial One

`ConfigureAwait(false)` is one of those topics that generates heated debates. The short version: it tells the awaiter not to capture the synchronization context, which can prevent deadlocks in library code.

```csharp
// In a library
public async Task<string> GetDataAsync()
{
    var response = await httpClient.GetAsync(url).ConfigureAwait(false);
    return await response.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

The rule of thumb: use `ConfigureAwait(false)` in library code that doesn't need the original context. In application code (like ASP.NET Core controllers), you usually don't need it because ASP.NET Core doesn't have a synchronization context.

But here's the nuance: even in ASP.NET Core, if you're doing CPU-bound work after an await, `ConfigureAwait(false)` can improve performance by avoiding unnecessary thread switches. The performance difference is usually negligible, but in hot paths it can matter.

## TaskCompletionSource: The Escape Hatch

Sometimes you need to bridge between async and sync code, or create a `Task` that you complete manually. That's what `TaskCompletionSource` is for:

```csharp
public class AsyncEvent<T>
{
    private readonly List<TaskCompletionSource<T>> _waiters = new();
    private readonly object _lock = new();

    public Task<T> WaitAsync(CancellationToken cancellationToken = default)
    {
        var tcs = new TaskCompletionSource<T>();
        
        cancellationToken.Register(() => tcs.TrySetCanceled(cancellationToken));
        
        lock (_lock)
        {
            _waiters.Add(tcs);
        }

        return tcs.Task;
    }

    public void SetResult(T result)
    {
        List<TaskCompletionSource<T>> toComplete;
        lock (_lock)
        {
            toComplete = _waiters.ToList();
            _waiters.Clear();
        }

        foreach (var tcs in toComplete)
        {
            tcs.TrySetResult(result);
        }
    }
}
```

This pattern is useful for converting event-based APIs to async/await. But be careful: `TaskCompletionSource` tasks don't automatically handle exceptions or cancellation the way normal async methods do. You have to handle those cases explicitly.

## ValueTask: When Task Is Too Heavy

`Task` is a class, which means heap allocation. For hot paths that usually complete synchronously, this can cause GC pressure. `ValueTask` is a struct that can represent either a `Task` or a synchronous result:

```csharp
public ValueTask<int> GetCachedValueAsync(int key)
{
    if (_cache.TryGetValue(key, out var value))
    {
        return new ValueTask<int>(value); // Synchronous completion, no allocation
    }

    return new ValueTask<int>(LoadValueAsync(key)); // Asynchronous, allocates Task
}

private async Task<int> LoadValueAsync(int key)
{
    // ... async work
}
```

The rule: use `ValueTask` for methods that frequently complete synchronously (like cached lookups). Use `Task` for everything else. `ValueTask` has more overhead when it does need to be asynchronous, so don't use it everywhere.

Also, you can only await a `ValueTask` once. If you need to await it multiple times, call `.AsTask()` to convert it to a `Task`.

## Parallel Async Operations

When you have multiple independent async operations, you usually want to run them in parallel:

```csharp
// Wrong - runs sequentially
var user = await GetUserAsync(userId);
var posts = await GetPostsAsync(userId);
var comments = await GetCommentsAsync(userId);

// Right - runs in parallel
var userTask = GetUserAsync(userId);
var postsTask = GetPostsAsync(userId);
var commentsTask = GetCommentsAsync(userId);

await Task.WhenAll(userTask, postsTask, commentsTask);

var user = await userTask;
var posts = await postsTask;
var comments = await commentsTask;
```

Or more concisely:

```csharp
var (user, posts, comments) = await (
    GetUserAsync(userId),
    GetPostsAsync(userId),
    GetCommentsAsync(userId)
);
```

Wait, that last one doesn't work in C# yet. But you can use a helper:

```csharp
public static class TaskExtensions
{
    public static async Task<(T1, T2, T3)> WhenAll<T1, T2, T3>(
        Task<T1> task1, Task<T2> task2, Task<T3> task3)
    {
        await Task.WhenAll(task1, task2, task3);
        return (await task1, await task2, await task3);
    }
}
```

### Task.WhenAll vs Task.WhenAny

`Task.WhenAll` waits for all tasks to complete. `Task.WhenAny` waits for the first one to complete:

```csharp
// Wait for the first successful result from multiple sources
var tasks = new[]
{
    FetchFromPrimarySourceAsync(),
    FetchFromSecondarySourceAsync(),
    FetchFromTertiarySourceAsync()
};

var completedTask = await Task.WhenAny(tasks);
var result = await completedTask;

// Cancel the others (if they support cancellation)
// Note: This doesn't actually cancel them, just ignores the results
```

But here's the gotcha: `Task.WhenAny` doesn't cancel the other tasks. They keep running in the background. If you want to cancel them, you need to pass cancellation tokens and cancel the source when one completes.

## Async Enumerables: The Forgotten Feature

`IAsyncEnumerable<T>` lets you create async sequences. It's perfect for streaming data:

```csharp
public async IAsyncEnumerable<DataChunk> StreamDataAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    var offset = 0;
    const int chunkSize = 1000;

    while (!cancellationToken.IsCancellationRequested)
    {
        var chunk = await FetchChunkAsync(offset, chunkSize, cancellationToken);
        
        if (chunk == null || chunk.Count == 0)
            yield break;

        yield return chunk;
        offset += chunkSize;
    }
}

// Usage
await foreach (var chunk in StreamDataAsync(cancellationToken))
{
    await ProcessChunkAsync(chunk, cancellationToken);
}
```

The `[EnumeratorCancellation]` attribute is important-it allows the cancellation token passed to `GetAsyncEnumerator` to be used, not just the one passed to the method.

## Deadlock Prevention

The classic deadlock scenario:

```csharp
// DON'T DO THIS
public string GetData()
{
    return GetDataAsync().Result; // Deadlock risk!
}
```

If `GetDataAsync` tries to resume on the UI thread (or any synchronization context), and `GetData` is blocking that thread waiting for the result, you have a deadlock.

Solutions:
1. Make the calling method async: `public async Task<string> GetDataAsync()`
2. Use `ConfigureAwait(false)` in the async method (if you control it)
3. Use `GetAwaiter().GetResult()` instead of `.Result` (slightly better, but still not great)
4. Just don't block on async code. Ever.

## Exception Handling in Async Code

Exceptions in async methods work mostly like sync methods, but there are quirks:

```csharp
public async Task ProcessAsync()
{
    try
    {
        await Operation1Async();
        await Operation2Async();
    }
    catch (SpecificException ex)
    {
        // Handle specific exception
        await LogErrorAsync(ex);
        throw; // Re-throw to preserve stack trace
    }
}
```

But if you're using `Task.WhenAll`, exceptions are wrapped in an `AggregateException`:

```csharp
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (AggregateException ex)
{
    // Multiple exceptions
    foreach (var inner in ex.InnerExceptions)
    {
        // Handle each
    }
}
```

Actually, wait-that's not quite right. When you await a `Task` that faults, the first exception is thrown directly, not wrapped. `AggregateException` only appears when you access `Task.Exception` directly or use certain `Task` methods.

To get all exceptions from `Task.WhenAll`:

```csharp
var tasks = new[] { task1, task2, task3 };
await Task.WhenAll(tasks);

// Check each task for exceptions
foreach (var task in tasks)
{
    if (task.IsFaulted)
    {
        // Handle task.Exception
    }
}
```

## Performance: Async State Machines

Every `async` method becomes a state machine class. This has overhead:
- Heap allocation for the state machine
- Boxing of value types in some cases
- Method call overhead

For hot paths, this can matter. But the alternative (blocking threads) is usually worse. Only optimize if profiling shows it's a problem.

One optimization: if a method can complete synchronously, return a completed task:

```csharp
public Task<string> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
    {
        return Task.FromResult(value); // No async state machine
    }

    return GetValueAsync(key); // Actually async
}
```

## Best Practices

1. **Always pass CancellationToken** - Make it a habit. Your future self will thank you.

2. **Use ConfigureAwait(false) in libraries** - But don't obsess over it in application code.

3. **Don't block on async code** - Use async all the way, or use `GetAwaiter().GetResult()` as a last resort.

4. **Handle exceptions appropriately** - Remember that `Task.WhenAll` doesn't wrap exceptions when awaited.

5. **Use ValueTask for hot synchronous paths** - But measure first.

6. **Parallelize independent operations** - Don't await sequentially when you can await in parallel.

7. **Consider IAsyncEnumerable for streaming** - It's more efficient than loading everything into memory.

## Conclusion

Async/await in C# is deceptively simple. The syntax is easy, but the semantics are complex. Understanding cancellation, proper exception handling, and performance characteristics is what separates developers who write async code from developers who write *correct* async code.

The key is to think about async operations as first-class citizens in your design, not as an afterthought. Plan for cancellation, handle exceptions properly, and understand the performance implications. Your production systems will thank you.

And remember: when in doubt, make it async and pass a CancellationToken. You can always optimize later if profiling shows it's necessary.

