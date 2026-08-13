# Chapter 20 — Real-Time Updates

## Design question

> A WPF dashboard receives thousands of server updates per second. Design it without freezing the UI.

## Architecture

```text
Server
  |
WebSocket / SignalR
  |
Background Receiver
  |
Channel / Buffer
  |
Batch Processor
  |
UI Dispatcher
  |
ViewModel
  |
Virtualized UI
```

## Channel

```csharp
private readonly Channel<CustomerUpdate> _updates =
    Channel.CreateBounded<CustomerUpdate>(
        new BoundedChannelOptions(10_000)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true,
            SingleWriter = false
        });
```

## Producer

```csharp
public async Task ReceiveAsync(
    CustomerUpdate update)
{
    await _updates.Writer.WriteAsync(update);
}
```

## Consumer

```csharp
public async Task ProcessUpdatesAsync(
    CancellationToken token)
{
    var batch = new List<CustomerUpdate>(100);

    while (await _updates.Reader.WaitToReadAsync(token))
    {
        batch.Clear();

        while (batch.Count < 100 &&
               _updates.Reader.TryRead(out var update))
        {
            batch.Add(update);
        }

        if (batch.Count > 0)
            await ApplyBatchToUiAsync(batch);
    }
}
```

## UI batching

Do not do:

```text
Update 1 -> Dispatcher
Update 2 -> Dispatcher
Update 3 -> Dispatcher
...
Update 10,000 -> Dispatcher
```

Instead:

```text
10,000 updates
      |
buffer
      |
batch
      |
one UI update
```

## Design concerns

Define:
- maximum buffer size
- backpressure/drop policy
- ordering requirements
- reconnect strategy
- duplicate handling
- UI update frequency
- stale data handling

For dashboards, coalescing updates can be more important than displaying every intermediate state.
