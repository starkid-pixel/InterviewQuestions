# Chapter 7 — Async and Background Processing

## Design question

> Process 100,000 items without freezing the WPF UI.

## Architecture

```text
UI
 |
Async Command
 |
Work Coordinator
 |
Bounded Worker Pool
 |
CPU / I/O Work
 |
Progress + Cancellation
 |
ViewModel
```

## Worker coordinator

```csharp
public sealed class WorkCoordinator
{
    public async Task ProcessAsync(
        IReadOnlyList<string> items,
        int maxConcurrency,
        IProgress<int>? progress,
        CancellationToken cancellationToken)
    {
        var completed = 0;

        await Parallel.ForEachAsync(
            items,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = maxConcurrency,
                CancellationToken = cancellationToken
            },
            async (item, token) =>
            {
                await ProcessItemAsync(item, token);

                var count =
                    Interlocked.Increment(ref completed);

                progress?.Report(count);
            });
    }

    private Task ProcessItemAsync(
        string item,
        CancellationToken token)
    {
        // CPU or async I/O operation.
        return Task.CompletedTask;
    }
}
```

## ViewModel

```csharp
private CancellationTokenSource? _cts;

public async Task StartAsync()
{
    _cts = new CancellationTokenSource();

    try
    {
        var progress = new Progress<int>(
            value => Completed = value);

        await _coordinator.ProcessAsync(
            Items,
            maxConcurrency: Environment.ProcessorCount,
            progress,
            _cts.Token);
    }
    catch (OperationCanceledException)
    {
    }
    finally
    {
        _cts.Dispose();
        _cts = null;
    }
}

public void Cancel()
{
    _cts?.Cancel();
}
```

## Important design points

Do not create 100,000 uncontrolled `Task.Run` calls.

Use bounded concurrency.

For CPU-bound work, a bounded worker pool prevents excessive ThreadPool pressure. For asynchronous I/O, use asynchronous APIs and limit concurrency according to server/resource constraints.

## Progress

`Progress<T>` normally captures the current synchronization context when created on the UI thread, making UI updates convenient.

## Cancellation

Cancellation must be cooperative. A token only requests cancellation; the operation must observe it.
