# Chapter 21 — Application Shutdown Design

## Design question

> How do you shut down a WPF application safely when API calls and background jobs are running?

## Shutdown sequence

```text
User closes application
        |
Stop accepting new work
        |
Cancel background operations
        |
Await graceful completion
        |
Persist required state
        |
Dispose services
        |
Application exits
```

## Coordinator

```csharp
public interface IApplicationLifetime
{
    Task ShutdownAsync();
}
```

```csharp
public sealed class ApplicationLifetime
    : IApplicationLifetime
{
    private readonly CancellationTokenSource _shutdown =
        new();

    private readonly IWorker _worker;
    private readonly IStateStore _state;

    public ApplicationLifetime(
        IWorker worker,
        IStateStore state)
    {
        _worker = worker;
        _state = state;
    }

    public async Task ShutdownAsync()
    {
        _shutdown.Cancel();

        await _worker.StopAsync();

        await _state.SaveAsync();
    }
}
```

## Window closing

```csharp
private async void Window_Closing(
    object? sender,
    CancelEventArgs e)
{
    e.Cancel = true;

    await _lifetime.ShutdownAsync();

    Closing -= Window_Closing;
    Close();
}
```

In a production implementation, guard against repeated close events and ensure exceptions during shutdown are handled safely.

## Unsaved changes

Before closing:
1. detect dirty state
2. ask user
3. save/discard/cancel
4. only then continue shutdown

## Interview point

Shutdown is a lifecycle problem. Every long-running service should have an explicit ownership and cancellation strategy.
