You now have the **tactics under Control Resource Demand**:

```
```

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

# 1. Manage Sampling Rate

## What problem does it solve?

The system receives data or events **too frequently**.

```
```

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
```

```
1,000 events / second
        ↓
Throttle
        ↓
Process 10 times / second
```

Example:

```
```

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
```

```
Events:

1 2 3 4 5 6 7 8 9 10

Process:

1       5       10
```

Example:

```
```

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
```

```
Incoming data continuously
```

sample the latest value periodically:

```
```

```
Latest Value
     ↓
Every 500 ms
     ↓
Process
```

Example:

```
```

```
Temperature:
20 → 21 → 22 → 23 → 24

Every 1 second:
Process latest value = 24
```

### Main mapping

```
```

```
Manage Sampling Rate
        ↓
Throttling
Downsampling
Periodic Sampling
Latest-value sampling
```

---

# 2. Limit Event Response

## What problem does it solve?

Too many events cause too many reactions.

```
```

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
```

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
```

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
```

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
```

```
MouseMoved
MouseMoved
MouseMoved
```

If the mouse position change is insignificant:

```
```

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
```

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

# 3. Prioritize Events

## What problem does it solve?

Not all work is equally important.

Imagine:

```
```

```
Critical Error
Telemetry
User Click
Background Sync
```

If everything is processed equally:

```
```

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
```

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
```

```
PriorityQueue<WorkItem, int> queue = new();

queue.Enqueue(telemetry, 3);
queue.Enqueue(userAction, 1);
queue.Enqueue(criticalError, 0);
```

Lower priority value:

```
```

```
0 → Highest
1 → High
2 → Normal
3 → Low
```

---

### 2. Separate Queues

```
```

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
```

```
System Capacity = 100 events/sec

Incoming = 1,000 events/sec
```

The system may choose:

```
```

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
```

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

# 4. Reduce Overhead

## What problem does it solve?

Sometimes the real problem is not the amount of data itself.

The problem is the **cost of repeatedly doing small things**.

Example:

```
```

```
100 records
      ↓
100 network calls
```

Each request has overhead:

```
```

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
```

```
100 records
      ↓
1 batch request
```

## Implementation approaches

### 1. Batching

```
```

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
```

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
```

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
```

```
1,000,000 records available

User needs:

100 records
```

Retrieve:

```
```

```
Only 100 records
```

---

### 4. Avoid Duplicate Work

Example:

```
```

```
Request A → Load Customer 10
Request B → Load Customer 10
Request C → Load Customer 10
```

Instead:

```
```

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
```

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

# 5. Bound Execution Times

## What problem does it solve?

Some operations may take too long.

```
```

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
```

```
using var cts =
    new CancellationTokenSource(
        TimeSpan.FromSeconds(5));

await SomeOperationAsync(cts.Token);
```

Conceptually:

```
```

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
```

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
```

```
await GenerateReportAsync(cancellationToken);
```

---

### 3. Time Budget

Instead of:

```
```

```
Process forever
```

use:

```
```

```
Maximum 100 ms per UI cycle
```

For example:

```
```

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
```

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
```

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
```

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

# 6. Increase Resource Efficiency

## What problem does it solve?

The system may be doing the required work, but using resources inefficiently.

```
```

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
```

```
1 GB data
   ↓
Load everything
```

use:

```
```

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
```

```
Producer
   ↓
Data
   ↓
Consumer processes immediately
```

Instead of:

```
```

```
Receive everything
       ↓
Store everything
       ↓
Process everything
```

Use:

```
```

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
```

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
```

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
```

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
```

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
```

```
1,000 tasks
     ↓
CPU overload
Thread contention
Memory pressure
```

Instead:

```
```

```
Semaphore
    ↓
Maximum 10 concurrent operations
```

Example:

```
```

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
```

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
```

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
```

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
```

```
Tactic
   ↓
Approach
   ↓
Implementation
```

For example:

```
```

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
```

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
