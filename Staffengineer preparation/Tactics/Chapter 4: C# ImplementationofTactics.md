# Control Resource Demand — Complete Source-Preserved Reference

> **Performance Efficiency → Control Resource Demand → Tactic → Approach → Implementation**

This version is deliberately **source-preserving**. It combines the complete technology-independent material with the complete supplied C# implementation material. Nothing is silently invented to fill a gap.

Where the conceptual source names an approach but the supplied C# source does not contain an implementation for it, the gap is explicitly marked.

---

# Part I — Performance Efficiency / Control Resource Demand

You now have the **tactics under Control Resource Demand**:

```
Performance Efficiency
        ↓
Control Resource Demand
        ↓
├── Manage Sampling Rate
├── Limit Event Response
├── Prioritize Events
├── Reduce Overhead
├── Bound Execution Times
└── Increase Resource Efficiency
```

The key question is:

> **How do we actually implement each tactic?**

Below is a technology-independent view first, followed by concrete examples.

---

---

# 1. Manage Sampling Rate

## Technology-Independent Concept

## What problem does it solve?

The system receives data or events **too frequently**.

```
1,000 events / second
        ↓
System processes all 1,000
        ↓
High CPU / Network / UI usage
```

The idea is:

> **We do not need to process every event.**

## Implementation approaches

### 1. Throttling

Process at a fixed rate.

```
1,000 events / second
        ↓
Throttle
        ↓
Process 10 times / second
```

Example:

```
Events:     ● ● ● ● ● ● ● ● ● ● ●

Process:    ▲       ▲       ▲
```

Use cases:

-  Slider movement 
-  Mouse movement 
-  Scrolling 
-  Real-time dashboards 
-  Sensor data 

---

### 2. Downsampling

Take only some events.

```
Events:

1 2 3 4 5 6 7 8 9 10

Process:

1       5       10
```

Example:

```
if (eventNumber % 10 == 0)
{
    Process(data);
}
```

Useful for:

-  Telemetry 
-  Monitoring 
-  Sensor data 
-  Analytics 

---

### 3. Periodic Sampling

Instead of processing every update:

```
Incoming data continuously
```

sample the latest value periodically:

```
Latest Value
     ↓
Every 500 ms
     ↓
Process
```

Example:

```
Temperature:
20 → 21 → 22 → 23 → 24

Every 1 second:
Process latest value = 24
```

### Main mapping

```
Manage Sampling Rate
        ↓
Throttling
Downsampling
Periodic Sampling
Latest-value sampling
```

---

## Supplied C# Implementation Material

## Approach: Throttling

### Scenario

A slider generates many value-change events.

```
10 → 11 → 12 → 13 → 14 → 15 → ...
```

We don't want to perform expensive work for every value.

### C# Example

```
public sealed class Throttler
{
    private readonly TimeSpan _interval;
    private DateTime _lastExecution = DateTime.MinValue;

    public Throttler(TimeSpan interval)
    {
        _interval = interval;
    }

    public bool TryExecute(Action action)
    {
        var now = DateTime.UtcNow;

        if (now - _lastExecution < _interval)
            return false;

        _lastExecution = now;

        action();

        return true;
    }
}
```

Usage:

```
private readonly Throttler _throttler =
    new(TimeSpan.FromMilliseconds(500));

public void OnSliderChanged(int value)
{
    _throttler.TryExecute(() =>
    {
        Console.WriteLine(
            $"Processing value: {value}");
    });
}
```

---

## Approach: Downsampling

```
public sealed class DownSampler<T>
{
    private readonly int _rate;
    private int _count;

    public DownSampler(int rate)
    {
        _rate = rate;
    }

    public void Process(T item, Action<T> action)
    {
        _count++;

        if (_count % _rate == 0)
        {
            action(item);
        }
    }
}
```

Usage:

```
var sampler = new DownSampler<int>(10);

for (int i = 1; i <= 100; i++)
{
    sampler.Process(i, value =>
    {
        Console.WriteLine(
            $"Processing event {value}");
    });
}
```

Result:

```
10
20
30
40
...
100
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Throttling | Yes |
| Downsampling | Yes |
| Periodic Sampling | No — no matching implementation was supplied |
| Latest-value sampling | No — no matching implementation was supplied |

---

# 2. Limit Event Response

## Technology-Independent Concept

## What problem does it solve?

Too many events cause too many reactions.

```
Event 1 → Process
Event 2 → Process
Event 3 → Process
Event 4 → Process
...
```

The question becomes:

> **Do we really need to react to every event?**

## Implementation approaches

### 1. Debouncing

Wait until events stop.

```
Typing:

H → He → Hel → Hell → Hello

                        ↓
                  User stops
                        ↓
                     Wait
                        ↓
                    Search
```

Use cases:

-  Search box 
-  Auto-save 
-  Validation 
-  Filtering 

---

### 2. Event Coalescing

Combine multiple similar events into one.

```
Update 1
Update 2
Update 3
Update 4
     ↓
Combine
     ↓
One update
```

For example:

```
Window resized 100 times
        ↓
Keep latest size
        ↓
Perform one layout update
```

---

### 3. Ignore Unnecessary Events

Example:

```
MouseMoved
MouseMoved
MouseMoved
```

If the mouse position change is insignificant:

```
if (Math.Abs(newPosition - oldPosition) < threshold)
{
    return;
}
```

Only meaningful events are processed.

---

### Main mapping

```
Limit Event Response
        ↓
Debouncing
Event Coalescing
Filtering
Threshold-based response
Ignore duplicate events
```

---

## Supplied C# Implementation Material

## Approach: Debouncing

### Scenario

The user types:

```
H → He → Hel → Hell → Hello
```

We want to search only after the user stops typing.

### C# Implementation

```
public sealed class Debouncer
{
    private CancellationTokenSource? _cts;

    public async Task DebounceAsync(
        TimeSpan delay,
        Func<Task> action)
    {
        _cts?.Cancel();

        _cts = new CancellationTokenSource();

        var token = _cts.Token;

        try
        {
            await Task.Delay(delay, token);

            if (!token.IsCancellationRequested)
            {
                await action();
            }
        }
        catch (OperationCanceledException)
        {
            // Expected when another event arrives.
        }
    }
}
```

Usage:

```
private readonly Debouncer _debouncer = new();

public async Task OnSearchTextChangedAsync(
    string searchText)
{
    await _debouncer.DebounceAsync(
        TimeSpan.FromMilliseconds(500),
        async () =>
        {
            await SearchAsync(searchText);
        });
}
```

---

## Approach: Event Coalescing

Keep only the latest event.

```
public sealed class LatestValueProcessor<T>
{
    private T? _latestValue;

    public void Update(T value)
    {
        _latestValue = value;
    }

    public void Process(Action<T> action)
    {
        if (_latestValue is not null)
        {
            action(_latestValue);
        }
    }
}
```

Example:

```
private readonly LatestValueProcessor<int>
    _processor = new();

public void OnSliderChanged(int value)
{
    _processor.Update(value);
}
```

Later:

```
_processor.Process(value =>
{
    Console.WriteLine(
        $"Processing latest value: {value}");
});
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Debouncing | Yes |
| Event Coalescing | Yes |
| Ignore Unnecessary Events | No — no matching implementation was supplied |
| Filtering | No — no matching implementation was supplied |
| Threshold-based response | No — no matching implementation was supplied |
| Ignore duplicate events | No — no matching implementation was supplied |

---

# 3. Prioritize Events

## Technology-Independent Concept

## What problem does it solve?

Not all work is equally important.

Imagine:

```
Critical Error
Telemetry
User Click
Background Sync
```

If everything is processed equally:

```
FIFO Queue

Telemetry
Telemetry
Telemetry
Telemetry
Critical Error ← waits
```

That is a problem.

Instead:

```
Priority Queue

Critical Error
User Action
Important Update
Telemetry
Background Work
```

## Implementation approaches

### 1. Priority Queue

```
PriorityQueue<WorkItem, int> queue = new();

queue.Enqueue(telemetry, 3);
queue.Enqueue(userAction, 1);
queue.Enqueue(criticalError, 0);
```

Lower priority value:

```
0 → Highest
1 → High
2 → Normal
3 → Low
```

---

### 2. Separate Queues

```
High Priority Queue
        ↓
User interactions
Critical operations

Normal Queue
        ↓
Normal processing

Low Priority Queue
        ↓
Logging
Telemetry
Background tasks
```

The scheduler checks high-priority work first.

---

### 3. Backpressure / Dropping Low-Priority Work

If overloaded:

```
System Capacity = 100 events/sec

Incoming = 1,000 events/sec
```

The system may choose:

```
Keep:
✓ User actions
✓ Payments
✓ Critical messages

Drop / delay:
✗ Debug logs
✗ Telemetry
✗ Low-priority refreshes
```

### Main mapping

```
Prioritize Events
        ↓
Priority Queue
Separate Queues
Priority Scheduler
Drop Low-Priority Work
Delay Background Work
```

---

## Supplied C# Implementation Material

## Approach: Priority Queue

```
public enum WorkPriority
{
    Critical = 0,
    High = 1,
    Normal = 2,
    Low = 3
}

public record WorkItem(
    string Name,
    WorkPriority Priority,
    Func<Task> Execute);
```

Queue:

```
var queue =
    new PriorityQueue<WorkItem, int>();

queue.Enqueue(
    new WorkItem(
        "Telemetry",
        WorkPriority.Low,
        () => SendTelemetryAsync()),
    (int)WorkPriority.Low);

queue.Enqueue(
    new WorkItem(
        "UserAction",
        WorkPriority.High,
        () => ProcessUserActionAsync()),
    (int)WorkPriority.High);
```

Processing:

```
while (queue.Count > 0)
{
    var work = queue.Dequeue();

    await work.Execute();
}
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Priority Queue | Yes |
| Separate Queues | No — no matching implementation was supplied |
| Backpressure / Dropping Low-Priority Work | No — no matching implementation was supplied |
| Priority Scheduler | No — no matching implementation was supplied |
| Drop Low-Priority Work | No — no matching implementation was supplied |
| Delay Background Work | No — no matching implementation was supplied |

---

# 4. Reduce Overhead

## Technology-Independent Concept

## What problem does it solve?

Sometimes the real problem is not the amount of data itself.

The problem is the **cost of repeatedly doing small things**.

Example:

```
100 records
      ↓
100 network calls
```

Each request has overhead:

```
Connection
Serialization
Network
Authentication
Processing
Response
```

Instead:

```
100 records
      ↓
1 batch request
```

## Implementation approaches

### 1. Batching

```
Operation 1 ┐
Operation 2 │
Operation 3 ├── Batch → Process once
Operation 4 │
Operation 5 ┘
```

Use cases:

-  Database inserts 
-  API requests 
-  UI updates 
-  Logging 

---

### 2. Caching

Avoid repeating expensive work.

```
Request
   ↓
Cache?
   │
   ├── Yes → Return cached result
   │
   └── No
          ↓
      Expensive operation
          ↓
      Store in cache
```

Example:

```
if (_cache.TryGetValue(key, out var result))
{
    return result;
}

result = CalculateExpensiveValue();

_cache[key] = result;

return result;
```

---

### 3. Paging

Avoid retrieving unnecessary data.

```
1,000,000 records available

User needs:

100 records
```

Retrieve:

```
Only 100 records
```

---

### 4. Avoid Duplicate Work

Example:

```
Request A → Load Customer 10
Request B → Load Customer 10
Request C → Load Customer 10
```

Instead:

```
Load Customer 10 once
       ↓
Reuse result
```

This can be implemented using:

-  Cache 
-  Request deduplication 
-  Shared tasks/futures 
-  Memoization 

---

### Main mapping

```
Reduce Overhead
        ↓
Batching
Caching
Paging
Request Deduplication
Memoization
Reuse
Connection Pooling
Avoid Repeated Serialization
```

---

## Supplied C# Implementation Material

## Approach: Batching

Instead of:

```
Save → Save → Save → Save
```

We collect items:

```
Save
Save
Save
Save
  ↓
Batch
  ↓
Save all
```

### C# Implementation

```
public sealed class BatchProcessor<T>
{
    private readonly int _batchSize;
    private readonly List<T> _buffer = new();

    public BatchProcessor(int batchSize)
    {
        _batchSize = batchSize;
    }

    public async Task AddAsync(
        T item,
        Func<IReadOnlyList<T>, Task> processBatch)
    {
        _buffer.Add(item);

        if (_buffer.Count < _batchSize)
            return;

        var batch = _buffer.ToList();

        _buffer.Clear();

        await processBatch(batch);
    }
}
```

Usage:

```
var processor =
    new BatchProcessor<Customer>(100);

foreach (var customer in customers)
{
    await processor.AddAsync(
        customer,
        async batch =>
        {
            await SaveCustomersAsync(batch);
        });
}
```

---

## Approach: Caching

```
public sealed class CustomerCache
{
    private readonly Dictionary<int, Customer>
        _cache = new();

    public async Task<Customer> GetAsync(
        int customerId,
        Func<int, Task<Customer>> factory)
    {
        if (_cache.TryGetValue(
            customerId,
            out var customer))
        {
            return customer;
        }

        customer = await factory(customerId);

        _cache[customerId] = customer;

        return customer;
    }
}
```

Usage:

```
var customer =
    await _cache.GetAsync(
        customerId,
        _repository.GetAsync);
```

---

## Approach: Paging

```
public static class PagingExtensions
{
    public static IEnumerable<T> GetPage<T>(
        this IEnumerable<T> source,
        int pageNumber,
        int pageSize)
    {
        return source
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize);
    }
}
```

Usage:

```
var page = customers.GetPage(
    pageNumber: 2,
    pageSize: 100);
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Batching | Yes |
| Caching | Yes |
| Paging | Yes |
| Avoid Duplicate Work | No — no matching implementation was supplied |
| Request Deduplication | No — no matching implementation was supplied |
| Shared tasks/futures | No — no matching implementation was supplied |
| Memoization | No — no matching implementation was supplied |
| Reuse | No — no matching implementation was supplied |
| Connection Pooling | No — no matching implementation was supplied |
| Avoid Repeated Serialization | No — no matching implementation was supplied |

---

# 5. Bound Execution Times

## Technology-Independent Concept

## What problem does it solve?

Some operations may take too long.

```
Request
   ↓
Processing...
   ↓
Processing...
   ↓
Processing...
   ↓
Never returns?
```

One slow operation can consume resources for too long.

The tactic says:

> **There must be a limit.**

## Implementation approaches

### 1. Timeout

```
using var cts =
    new CancellationTokenSource(
        TimeSpan.FromSeconds(5));

await SomeOperationAsync(cts.Token);
```

Conceptually:

```
Operation
    ↓
Maximum = 5 seconds
    │
    ├── Complete → Success
    │
    └── Too long → Cancel / Fail
```

---

### 2. Cancellation

Allow work to stop.

```
User starts report generation
          ↓
User cancels
          ↓
CancellationToken
          ↓
Stop processing
```

Example:

```
await GenerateReportAsync(cancellationToken);
```

---

### 3. Time Budget

Instead of:

```
Process forever
```

use:

```
Maximum 100 ms per UI cycle
```

For example:

```
Start processing
       ↓
100 ms elapsed?
       │
       ├── No → Continue
       │
       └── Yes → Stop / Yield
```

---

### 4. Circuit Breaker

If a dependency repeatedly fails or takes too long:

```
Application
     ↓
Slow/Failing Service
     ↓
Slow
     ↓
Slow
     ↓
Slow
```

Stop sending requests temporarily:

```
Circuit Open
     ↓
Fail Fast
     ↓
Wait
     ↓
Try Again
```

### Main mapping

```
Bound Execution Times
        ↓
Timeout
Cancellation
Time Budget
Circuit Breaker
Fail Fast
```

---

## Supplied C# Implementation Material

## Approach: Timeout

```
public static async Task<T> ExecuteWithTimeout<T>(
    Func<CancellationToken, Task<T>> operation,
    TimeSpan timeout)
{
    using var cts =
        new CancellationTokenSource(timeout);

    return await operation(cts.Token);
}
```

Usage:

```
var result =
    await ExecuteWithTimeout(
        token => GetDataAsync(token),
        TimeSpan.FromSeconds(5));
```

Conceptually:

```
Operation
    ↓
5 seconds maximum
    │
    ├── Completed → Success
    │
    └── Too slow → Cancel
```

---

## Approach: Cancellation

```
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    foreach (var item in _items)
    {
        cancellationToken.ThrowIfCancellationRequested();

        await ProcessItemAsync(
            item,
            cancellationToken);
    }
}
```

The caller can stop the work:

```
using var cts = new CancellationTokenSource();

var task = ProcessAsync(cts.Token);

// Later
cts.Cancel();
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Timeout | Yes |
| Cancellation | Yes |
| Time Budget | No — no matching implementation was supplied |
| Circuit Breaker | No — no matching implementation was supplied |
| Fail Fast | No — no matching implementation was supplied |

---

# 6. Increase Resource Efficiency

## Technology-Independent Concept

## What problem does it solve?

The system may be doing the required work, but using resources inefficiently.

```
Same work
   ↓
Too much memory
Too much CPU
Too many threads
Too many connections
```

The goal is:

> **Do the same or required amount of work using resources more efficiently.**

## Implementation approaches

### 1. Chunking

Instead of:

```
1 GB data
   ↓
Load everything
```

use:

```
10 MB
 ↓
Process
 ↓
Next 10 MB
```

---

### 2. Streaming

Process data as it arrives.

```
Producer
   ↓
Data
   ↓
Consumer processes immediately
```

Instead of:

```
Receive everything
       ↓
Store everything
       ↓
Process everything
```

Use:

```
Receive
   ↓
Process
   ↓
Receive
   ↓
Process
```

---

### 3. Object Pooling

Instead of:

```
Create object
Use
Destroy

Create object
Use
Destroy
```

Reuse:

```
Pool
 ↓
Get object
 ↓
Use
 ↓
Return to pool
```

Useful for expensive objects.

---

### 4. Connection Pooling

Instead of:

```
Request
 ↓
Create database connection
 ↓
Use
 ↓
Destroy
```

Use:

```
Connection Pool
       ↓
Borrow connection
       ↓
Use
       ↓
Return connection
```

---

### 5. Concurrency Control

Too much concurrency can reduce performance:

```
1,000 tasks
     ↓
CPU overload
Thread contention
Memory pressure
```

Instead:

```
Semaphore
    ↓
Maximum 10 concurrent operations
```

Example:

```
var semaphore = new SemaphoreSlim(10);

await semaphore.WaitAsync();

try
{
    await ProcessAsync();
}
finally
{
    semaphore.Release();
}
```

### Main mapping

```
Increase Resource Efficiency
        ↓
Chunking
Streaming
Pooling
Connection Reuse
Concurrency Control
Reuse Buffers
Memory Management
```

---

# Complete Summary

```
CONTROL RESOURCE DEMAND
│
├── 1. Manage Sampling Rate
│       │
│       ├── Throttling
│       ├── Downsampling
│       ├── Periodic Sampling
│       └── Latest-value Sampling
│
├── 2. Limit Event Response
│       │
│       ├── Debouncing
│       ├── Event Coalescing
│       ├── Event Filtering
│       └── Threshold-based Processing
│
├── 3. Prioritize Events
│       │
│       ├── Priority Queue
│       ├── Separate Queues
│       ├── Priority Scheduler
│       ├── Delay Low-Priority Work
│       └── Drop Low-Priority Work
│
├── 4. Reduce Overhead
│       │
│       ├── Batching
│       ├── Caching
│       ├── Paging
│       ├── Request Deduplication
│       ├── Memoization
│       └── Reuse
│
├── 5. Bound Execution Times
│       │
│       ├── Timeout
│       ├── Cancellation
│       ├── Time Budget
│       ├── Circuit Breaker
│       └── Fail Fast
│
└── 6. Increase Resource Efficiency
        │
        ├── Chunking
        ├── Streaming
        ├── Object Pooling
        ├── Connection Pooling
        ├── Concurrency Control
        └── Buffer Reuse
```

## The most important thing to remember

These are **not implementations themselves**:

```
Manage Sampling Rate
Limit Event Response
Prioritize Events
Reduce Overhead
Bound Execution Times
Increase Resource Efficiency
```

They are **tactics**.

Under each tactic, you choose an **approach**:

```
Tactic
   ↓
Approach
   ↓
Implementation
```

For example:

```
Performance
    ↓
Control Resource Demand
    ↓
Manage Sampling Rate
    ↓
Throttling
    ↓
Execute at most once every 500 ms
```

Or:

```
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Batching
    ↓
Collect 100 database writes
    ↓
Execute one bulk operation
```

This hierarchy is a very useful way to explain your design decisions in a **system design or architecture interview**.

## Supplied C# Implementation Material

## Approach: Chunking

```
public static async Task ProcessInChunksAsync<T>(
    IEnumerable<T> items,
    int chunkSize,
    Func<IEnumerable<T>, Task> process)
{
    foreach (var chunk in items.Chunk(chunkSize))
    {
        await process(chunk);
    }
}
```

Usage:

```
await ProcessInChunksAsync(
    customers,
    chunkSize: 1000,
    async chunk =>
    {
        await ProcessCustomersAsync(chunk);
    });
```

---

## Approach: Streaming

```
public async IAsyncEnumerable<Customer>
    GetCustomersAsync()
{
    foreach (var customer in _customers)
    {
        await Task.Delay(100);

        yield return customer;
    }
}
```

Consume:

```
await foreach (
    var customer in GetCustomersAsync())
{
    Process(customer);
}
```

Instead of:

```
Load everything
        ↓
Store everything
        ↓
Process everything
```

We do:

```
Receive
   ↓
Process
   ↓
Receive
   ↓
Process
```

---

## Approach: Concurrency Control

```
public async Task ProcessAsync<T>(
    IEnumerable<T> items,
    int maxConcurrency)
{
    using var semaphore =
        new SemaphoreSlim(maxConcurrency);

    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();

        try
        {
            await ProcessItemAsync(item);
        }
        finally
        {
            semaphore.Release();
        }
    });

    await Task.WhenAll(tasks);
}
```

Usage:

```
await ProcessAsync(
    customers,
    maxConcurrency: 10);
```

This prevents:

```
10,000 tasks
     ↓
Resource exhaustion
```

and instead gives:

```
10,000 items
     ↓
Maximum 10 active operations
     ↓
Controlled resource usage
```

---

## Coverage Check

The following records exactly which conceptual approaches have matching implementation material in the supplied C# source.

| Conceptual approach | Matching C# material supplied? |
|---|---|
| Chunking | Yes |
| Streaming | Yes |
| Object Pooling | No — no matching implementation was supplied |
| Connection Pooling | No — no matching implementation was supplied |
| Concurrency Control | Yes |
| Reuse Buffers | No — no matching implementation was supplied |
| Memory Management | No — no matching implementation was supplied |

---

# Appendix A — Complete C# Implementation Source

The following is the complete supplied implementation reference, preserved as a separate source section so that no implementation detail is lost.

Absolutely. That would make the material much more useful as a **reference and demo project**.

We can structure every tactic consistently like this:

```
Performance Efficiency
        ↓
Control Resource Demand
        ↓
Tactic
        ↓
Problem
        ↓
Approaches
        ↓
C# Implementation
        ↓
Before / After Behavior
```

For example:

---

# 1. Manage Sampling Rate

## Approach: Throttling

### Scenario

A slider generates many value-change events.

```
10 → 11 → 12 → 13 → 14 → 15 → ...
```

We don't want to perform expensive work for every value.

### C# Example

```
public sealed class Throttler
{
    private readonly TimeSpan _interval;
    private DateTime _lastExecution = DateTime.MinValue;

    public Throttler(TimeSpan interval)
    {
        _interval = interval;
    }

    public bool TryExecute(Action action)
    {
        var now = DateTime.UtcNow;

        if (now - _lastExecution < _interval)
            return false;

        _lastExecution = now;

        action();

        return true;
    }
}
```

Usage:

```
private readonly Throttler _throttler =
    new(TimeSpan.FromMilliseconds(500));

public void OnSliderChanged(int value)
{
    _throttler.TryExecute(() =>
    {
        Console.WriteLine(
            $"Processing value: {value}");
    });
}
```

---

## Approach: Downsampling

```
public sealed class DownSampler<T>
{
    private readonly int _rate;
    private int _count;

    public DownSampler(int rate)
    {
        _rate = rate;
    }

    public void Process(T item, Action<T> action)
    {
        _count++;

        if (_count % _rate == 0)
        {
            action(item);
        }
    }
}
```

Usage:

```
var sampler = new DownSampler<int>(10);

for (int i = 1; i <= 100; i++)
{
    sampler.Process(i, value =>
    {
        Console.WriteLine(
            $"Processing event {value}");
    });
}
```

Result:

```
10
20
30
40
...
100
```

---

# 2. Limit Event Response

## Approach: Debouncing

### Scenario

The user types:

```
H → He → Hel → Hell → Hello
```

We want to search only after the user stops typing.

### C# Implementation

```
public sealed class Debouncer
{
    private CancellationTokenSource? _cts;

    public async Task DebounceAsync(
        TimeSpan delay,
        Func<Task> action)
    {
        _cts?.Cancel();

        _cts = new CancellationTokenSource();

        var token = _cts.Token;

        try
        {
            await Task.Delay(delay, token);

            if (!token.IsCancellationRequested)
            {
                await action();
            }
        }
        catch (OperationCanceledException)
        {
            // Expected when another event arrives.
        }
    }
}
```

Usage:

```
private readonly Debouncer _debouncer = new();

public async Task OnSearchTextChangedAsync(
    string searchText)
{
    await _debouncer.DebounceAsync(
        TimeSpan.FromMilliseconds(500),
        async () =>
        {
            await SearchAsync(searchText);
        });
}
```

---

## Approach: Event Coalescing

Keep only the latest event.

```
public sealed class LatestValueProcessor<T>
{
    private T? _latestValue;

    public void Update(T value)
    {
        _latestValue = value;
    }

    public void Process(Action<T> action)
    {
        if (_latestValue is not null)
        {
            action(_latestValue);
        }
    }
}
```

Example:

```
private readonly LatestValueProcessor<int>
    _processor = new();

public void OnSliderChanged(int value)
{
    _processor.Update(value);
}
```

Later:

```
_processor.Process(value =>
{
    Console.WriteLine(
        $"Processing latest value: {value}");
});
```

---

# 3. Prioritize Events

## Approach: Priority Queue

```
public enum WorkPriority
{
    Critical = 0,
    High = 1,
    Normal = 2,
    Low = 3
}

public record WorkItem(
    string Name,
    WorkPriority Priority,
    Func<Task> Execute);
```

Queue:

```
var queue =
    new PriorityQueue<WorkItem, int>();

queue.Enqueue(
    new WorkItem(
        "Telemetry",
        WorkPriority.Low,
        () => SendTelemetryAsync()),
    (int)WorkPriority.Low);

queue.Enqueue(
    new WorkItem(
        "UserAction",
        WorkPriority.High,
        () => ProcessUserActionAsync()),
    (int)WorkPriority.High);
```

Processing:

```
while (queue.Count > 0)
{
    var work = queue.Dequeue();

    await work.Execute();
}
```

---

# 4. Reduce Overhead

## Approach: Batching

Instead of:

```
Save → Save → Save → Save
```

We collect items:

```
Save
Save
Save
Save
  ↓
Batch
  ↓
Save all
```

### C# Implementation

```
public sealed class BatchProcessor<T>
{
    private readonly int _batchSize;
    private readonly List<T> _buffer = new();

    public BatchProcessor(int batchSize)
    {
        _batchSize = batchSize;
    }

    public async Task AddAsync(
        T item,
        Func<IReadOnlyList<T>, Task> processBatch)
    {
        _buffer.Add(item);

        if (_buffer.Count < _batchSize)
            return;

        var batch = _buffer.ToList();

        _buffer.Clear();

        await processBatch(batch);
    }
}
```

Usage:

```
var processor =
    new BatchProcessor<Customer>(100);

foreach (var customer in customers)
{
    await processor.AddAsync(
        customer,
        async batch =>
        {
            await SaveCustomersAsync(batch);
        });
}
```

---

## Approach: Caching

```
public sealed class CustomerCache
{
    private readonly Dictionary<int, Customer>
        _cache = new();

    public async Task<Customer> GetAsync(
        int customerId,
        Func<int, Task<Customer>> factory)
    {
        if (_cache.TryGetValue(
            customerId,
            out var customer))
        {
            return customer;
        }

        customer = await factory(customerId);

        _cache[customerId] = customer;

        return customer;
    }
}
```

Usage:

```
var customer =
    await _cache.GetAsync(
        customerId,
        _repository.GetAsync);
```

---

## Approach: Paging

```
public static class PagingExtensions
{
    public static IEnumerable<T> GetPage<T>(
        this IEnumerable<T> source,
        int pageNumber,
        int pageSize)
    {
        return source
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize);
    }
}
```

Usage:

```
var page = customers.GetPage(
    pageNumber: 2,
    pageSize: 100);
```

---

# 5. Bound Execution Times

## Approach: Timeout

```
public static async Task<T> ExecuteWithTimeout<T>(
    Func<CancellationToken, Task<T>> operation,
    TimeSpan timeout)
{
    using var cts =
        new CancellationTokenSource(timeout);

    return await operation(cts.Token);
}
```

Usage:

```
var result =
    await ExecuteWithTimeout(
        token => GetDataAsync(token),
        TimeSpan.FromSeconds(5));
```

Conceptually:

```
Operation
    ↓
5 seconds maximum
    │
    ├── Completed → Success
    │
    └── Too slow → Cancel
```

---

## Approach: Cancellation

```
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    foreach (var item in _items)
    {
        cancellationToken.ThrowIfCancellationRequested();

        await ProcessItemAsync(
            item,
            cancellationToken);
    }
}
```

The caller can stop the work:

```
using var cts = new CancellationTokenSource();

var task = ProcessAsync(cts.Token);

// Later
cts.Cancel();
```

---

# 6. Increase Resource Efficiency

## Approach: Chunking

```
public static async Task ProcessInChunksAsync<T>(
    IEnumerable<T> items,
    int chunkSize,
    Func<IEnumerable<T>, Task> process)
{
    foreach (var chunk in items.Chunk(chunkSize))
    {
        await process(chunk);
    }
}
```

Usage:

```
await ProcessInChunksAsync(
    customers,
    chunkSize: 1000,
    async chunk =>
    {
        await ProcessCustomersAsync(chunk);
    });
```

---

## Approach: Streaming

```
public async IAsyncEnumerable<Customer>
    GetCustomersAsync()
{
    foreach (var customer in _customers)
    {
        await Task.Delay(100);

        yield return customer;
    }
}
```

Consume:

```
await foreach (
    var customer in GetCustomersAsync())
{
    Process(customer);
}
```

Instead of:

```
Load everything
        ↓
Store everything
        ↓
Process everything
```

We do:

```
Receive
   ↓
Process
   ↓
Receive
   ↓
Process
```

---

## Approach: Concurrency Control

```
public async Task ProcessAsync<T>(
    IEnumerable<T> items,
    int maxConcurrency)
{
    using var semaphore =
        new SemaphoreSlim(maxConcurrency);

    var tasks = items.Select(async item =>
    {
        await semaphore.WaitAsync();

        try
        {
            await ProcessItemAsync(item);
        }
        finally
        {
            semaphore.Release();
        }
    });

    await Task.WhenAll(tasks);
}
```

Usage:

```
await ProcessAsync(
    customers,
    maxConcurrency: 10);
```

This prevents:

```
10,000 tasks
     ↓
Resource exhaustion
```

and instead gives:

```
10,000 items
     ↓
Maximum 10 active operations
     ↓
Controlled resource usage
```

---

# Recommended structure for the complete reference

I would organize the final Markdown/reference document like this:

```
Performance Efficiency
│
└── Control Resource Demand
    │
    ├── 1. Manage Sampling Rate
    │       ├── What problem does it solve?
    │       ├── Throttling
    │       │       ├── Concept
    │       │       ├── Diagram
    │       │       ├── C# Code
    │       │       └── Use Cases
    │       │
    │       ├── Downsampling
    │       └── Periodic Sampling
    │
    ├── 2. Limit Event Response
    │       ├── Debouncing
    │       ├── Event Coalescing
    │       ├── Filtering
    │       └── Threshold Processing
    │
    ├── 3. Prioritize Events
    │       ├── Priority Queue
    │       ├── Multiple Queues
    │       └── Priority Scheduler
    │
    ├── 4. Reduce Overhead
    │       ├── Batching
    │       ├── Caching
    │       ├── Paging
    │       └── Request Deduplication
    │
    ├── 5. Bound Execution Times
    │       ├── Timeout
    │       ├── Cancellation
    │       ├── Time Budget
    │       └── Fail Fast
    │
    └── 6. Increase Resource Efficiency
            ├── Chunking
            ├── Streaming
            ├── Object Pooling
            ├── Connection Pooling
            └── Concurrency Control
```

This would be a strong **architecture reference** because it doesn't just say *"use caching, batching, throttling"*; it shows the full reasoning:

```
Problem
   ↓
Why it happens
   ↓
Performance tactic
   ↓
Possible approaches
   ↓
C# implementation
   ↓
Trade-offs
```

That is exactly the level at which you can explain these concepts in a longer system-design discussion.

---

# Appendix B — Final Learning Hierarchy

```text
Performance Efficiency
        ↓
Control Resource Demand
        ↓
Tactic
        ↓
Problem
        ↓
Approach
        ↓
Implementation
```

The important distinction is:

> **Tactics describe what we want to achieve. Approaches describe how we can achieve it. Implementations show how a chosen approach is realized in a technology such as C#.**
