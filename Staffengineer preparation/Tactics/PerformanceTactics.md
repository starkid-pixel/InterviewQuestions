<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/0714dea8-d2c8-45d1-97ec-00b00aee626f" />



# Performance Efficiency and Control Resource Demand

## 1. Overview

This document explains the **Performance** quality attribute and the tactic category **Control Resource Demand** in a technology-independent way.

The structure used throughout is:

```text
Quality Attribute
    ↓
Tactic Category
    ↓
Tactic
    ↓
Possible Approaches
    ↓
Possible Implementation
```

Example:

```text
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Caching
    ↓
Store and reuse a previously computed result
```

---

# 2. Performance Efficiency

Performance efficiency is concerned with the ability of a system to provide acceptable performance while making effective use of available resources.

Typical questions include:

- How quickly does the system respond?
- How much work can it process?
- How many resources are required?
- What happens when demand increases?
- Can the system maintain acceptable performance under load?

Resources may include:

- CPU
- Memory
- Network bandwidth
- Disk or storage I/O
- Database connections
- Threads or workers
- Processing time
- Other limited system resources

```text
Incoming Demand
       │
       ▼
+-------------------+
|      System       |
| CPU               |
| Memory            |
| Network           |
| Storage           |
| Workers           |
+-------------------+
       │
       ▼
Response / Result
```

Every system has finite capacity. If demand exceeds the capacity of available resources, performance degrades.

---

# 3. What Happens If We Do Not Control Resource Demand?

Every request, event, message, or operation requires work.

```text
Event
  │
  ├── CPU processing
  ├── Memory allocation
  ├── Network communication
  ├── Storage access
  └── Downstream processing
```

As demand increases:

```text
More Events / Requests
        ↓
More Work
        ↓
More Resource Consumption
```

A typical progression is:

```text
Increasing Demand
       ↓
Higher Resource Utilization
       ↓
Resource Saturation
       ↓
Waiting / Queueing
       ↓
Higher Response Time
       ↓
Timeouts / Failures
       ↓
Potential System Instability
```

## 3.1 CPU Saturation

Suppose a system can process:

```text
10,000 operations / second
```

but receives:

```text
50,000 operations / second
```

The excess work must wait, be rejected, or otherwise be controlled.

Possible consequences:

- Slower processing
- Increased waiting
- Increased response time
- Reduced throughput
- Competition between tasks

## 3.2 Queue Growth

When work arrives faster than it can be processed:

```text
Producer
50,000 events/sec
        │
        ▼
       Queue
        │
        ▼
Consumer
10,000 events/sec
```

The backlog can continue growing:

```text
Growing Queue
      ↓
Growing Memory Usage
      ↓
Memory Pressure
      ↓
Performance Degradation
      ↓
Possible Resource Exhaustion
```

## 3.3 Increased Response Time

A task may require only a small amount of actual processing time but spend a long time waiting for resources.

```text
Response Time
=
Waiting Time
+
Processing Time
```

As demand increases, waiting time can become a major cause of poor performance.

## 3.4 Memory Pressure

If the system retains all incoming work:

```text
Events
   ↓
Queue / Buffer
   ↓
Memory
```

then increasing demand can cause:

```text
More Data
    ↓
More Memory Allocation
    ↓
Memory Pressure
    ↓
More Resource Management Work
    ↓
Performance Degradation
    ↓
Possible Resource Exhaustion
```

## 3.5 Downstream Overload

Demand can propagate through the architecture:

```text
Client
   ↓
Service
   ↓
Database
   ↓
Storage
```

Excessive demand may create a feedback loop:

```text
High Demand
    ↓
Slow Responses
    ↓
Timeouts
    ↓
Retries
    ↓
Even Higher Demand
```

This can contribute to cascading failures.

---

# 4. Why Control Resource Demand?

One way to improve performance is to increase capacity or manage resources more effectively.

Another important strategy is to control the amount of work that demands those resources.

```text
                    Improve Performance
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
     Control the Demand           Improve Capacity /
        for Resources             Manage Resources
```

The key question is:

> Can we reduce or control the amount of work the system is required to perform?

Examples:

- Process fewer events.
- Process information less frequently.
- Avoid responding fully to every event.
- Process important work first.
- Avoid repeated work.
- Bound operations that consume resources for too long.
- Perform the same useful work more efficiently.

This leads to the tactic category:

# Control Resource Demand

Its purpose is:

> Control the amount of work that places demand on finite system resources.

---

# 5. Architectural Hierarchy

```text
Quality Attribute
        ↓
Performance
        ↓
Tactic Category
        ↓
Control Resource Demand
        ↓
Tactics
        ↓
Possible Approaches
        ↓
Possible Implementations
```

The tactics covered are:

```text
Control Resource Demand
│
├── Manage Sampling Rate
├── Limit Event Response
├── Prioritize Events
├── Reduce Overhead
├── Bound Execution Times
└── Increase Resource Efficiency
```

---

# 6. Tactic: Manage Sampling Rate

## What does it mean?

A system does not always need to collect or process information at the maximum available frequency.

The key question is:

> How often do we actually need to sample or process information?

```text
Data Source
     │
     ▼
Sampling Policy
     │
     ├── Every event
     ├── Every N events
     ├── Every interval
     └── Based on changing conditions
```

The goal is to control how frequently work is generated.

## Possible approaches

- Fixed-rate sampling
- Periodic sampling
- Sample every Nth event
- Adaptive sampling
- Dynamic sampling frequency
- Change-based sampling
- Aggregating multiple observations

## Possible implementation

```text
Receive data continuously
        │
        ▼
Should this data be sampled?
        │
   ┌────┴────┐
   │         │
  No        Yes
   │         │
Ignore     Process
```

Other examples:

```text
Process every 100th event
```

or:

```text
Process one sample every defined interval
```

The architectural idea is independent of the programming language or framework.

---

# 7. Tactic: Limit Event Response

## What does it mean?

An incoming event does not necessarily require a complete response every time.

```text
Event
   ↓
Process
   ↓
Update State
   ↓
Notify Other Components
   ↓
Perform Additional Work
```

The key question is:

> Does every event require the same amount of processing and response?

## Possible approaches

- Ignore duplicate events
- Ignore events that do not produce meaningful state changes
- Event filtering
- Event coalescing
- Event aggregation
- Batch event processing
- Throttling
- Debouncing
- Respond only when a meaningful condition occurs

## Possible implementation

```text
Incoming Events
       │
       ▼
Event Filter
       │
       ├── Duplicate → Ignore
       ├── No meaningful change → Ignore
       └── Important change → Process
```

Or:

```text
Many Events
     ↓
Collect / Combine
     ↓
One consolidated response
```

The objective is to reduce the amount of work triggered by event handling.

---

# 8. Tactic: Prioritize Events

## What does it mean?

Not every event has the same importance or urgency.

```text
Critical
High
Normal
Low
```

Under heavy demand, treating all work equally can cause important work to wait behind less important work.

The key question is:

> Which events should receive limited resources first?

## Possible approaches

- Priority classification
- Priority queues
- Severity levels
- Critical-first processing
- Deadline-based processing
- Different processing pipelines
- Deferring low-priority work
- Dropping low-value work under overload

## Possible implementation

```text
Incoming Events
       │
       ▼
Classify Priority
       │
 ┌─────┼────────┐
 ▼     ▼        ▼
High  Normal    Low
 │      │        │
 ▼      ▼        ▼
First  Later    Defer
```

Another model:

```text
High Priority Queue
Normal Priority Queue
Low Priority Queue
```

A scheduler selects work according to a defined policy.

---

# 9. Tactic: Reduce Overhead

## What does it mean?

Some work is necessary, but systems can also perform additional or repeated work that adds unnecessary cost.

Examples:

- Repeating the same computation
- Repeating network communication
- Repeated setup and teardown
- Processing duplicate information
- Repeated data conversion
- Repeated resource creation

The key question is:

> Can we perform the required work with less additional cost?

## Possible approaches

- Caching
- Batching
- Reuse
- Deduplication
- Avoid repeated computation
- Reduce unnecessary communication
- Reduce repeated setup
- Reduce unnecessary conversion or transformation

### Caching

```text
Request 1
    ↓
Compute
    ↓
Store Result

Request 2
    ↓
Reuse Result
```

### Batching

Instead of performing many small operations independently:

```text
Operation 1
Operation 2
Operation 3
```

an alternative is:

```text
Collect Operations
       ↓
Perform as Batch
```

### Reuse

```text
Need Resource
     ↓
Reuse Existing Resource
```

### Deduplication

```text
Same Work
Same Work
Same Work
       ↓
Detect Duplicate
       ↓
Perform Work Once
```

### Reduce Communication

```text
Request → Response
Request → Response
Request → Response
```

may be consolidated where appropriate.

## Possible implementation

```text
Caching
    → Store and reuse results

Batching
    → Accumulate work and process together

Reuse
    → Keep reusable resources available

Deduplication
    → Detect equivalent work before processing
```

---

# 10. Tactic: Bound Execution Times

## What does it mean?

An operation that consumes resources for an uncontrolled or unlimited amount of time can negatively affect the entire system.

```text
Operation Starts
       │
       ▼
Keeps Running
       │
       ▼
No Defined Limit
```

Such an operation may continue consuming:

- CPU
- Memory
- Threads or workers
- Connections
- Other limited resources

The key question is:

> How long should this operation be allowed to consume resources?

## Possible approaches

- Time limits
- Timeouts
- Cancellation
- Deadlines
- Maximum execution duration
- Maximum retry duration
- Maximum number of attempts

## Possible implementation

```text
Start Operation
       │
       ▼
Has the allowed execution time expired?
       │
   ┌───┴────┐
   │        │
  No       Yes
   │        │
Continue   Stop / Cancel / Fail
```

---

# 11. Tactic: Increase Resource Efficiency

## What does it mean?

This tactic focuses on obtaining more useful work from the same available resources.

The goal is not necessarily to reduce the number of incoming events. Instead, it reduces the cost of processing each unit of work.

The key question is:

> Can the system perform the same useful work while consuming fewer resources?

## Possible approaches

- More efficient algorithms
- Better data structures
- Reduce unnecessary processing steps
- Reduce unnecessary memory allocation
- Resource reuse
- Reduce unnecessary data movement
- Optimize communication
- Optimize storage access
- Avoid unnecessary copying

## Example

Before:

```text
1 Operation
    ↓
10 Processing Steps
```

After:

```text
1 Operation
    ↓
Only Necessary Processing Steps
```

The same useful result is produced while consuming fewer resources.

---

# 12. Paging: Where Does It Fit?

Paging divides a large data set into smaller portions and retrieves or processes only the portion currently required.

```text
Large Data Set
      │
      ▼
Page 1
Page 2
Page 3
...
```

Instead of:

```text
Load Entire Data Set
```

the system can:

```text
Load Only the Required Page
```

## What problem can paging solve?

Paging can reduce:

- Memory consumption
- Network transfer
- Serialization and deserialization work
- Processing of unnecessary data
- Initial response time

### Example: Reduce Overhead

```text
Request Entire Data Set
        ↓
Transfer Everything
        ↓
Process Everything
```

becomes:

```text
Request Required Page
        ↓
Transfer Required Data
        ↓
Process Required Data
```

In this context:

```text
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Paging
```

### Example: Limit Event Response

```text
User Opens Screen
       ↓
Load Entire Data Set
```

can become:

```text
User Opens Screen
       ↓
Load Only the Required Portion
```

Here paging limits the amount of work performed in response to the event.

Therefore:

> Paging is an approach whose classification depends on the problem it is solving.

---

# 13. Streaming: Where Does It Fit?

Streaming transfers or processes data progressively rather than waiting for the entire data set to be available.

```text
Chunk 1 → Process
Chunk 2 → Process
Chunk 3 → Process
...
```

Instead of:

```text
Receive Entire Data Set
        ↓
Store Entire Data Set
        ↓
Start Processing
```

streaming allows:

```text
Receive Small Portion
        ↓
Process
        ↓
Release / Reuse Resources
        ↓
Receive Next Portion
```

## What problem can streaming solve?

Streaming can reduce:

- Large memory requirements
- Large intermediate buffers
- Waiting for an entire data set before processing
- Large one-time transfers
- Unnecessary accumulation of data

For example:

```text
Very Large Data Set
        ↓
Do NOT hold all data in memory
        ↓
Process progressively
```

Depending on purpose:

```text
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Streaming / Chunked Processing
```

Streaming may also support other resource-related goals depending on the architectural reason for using it.

---

# 14. Paging vs Streaming

| Aspect | Paging | Streaming |
|---|---|---|
| Main idea | Divide data into discrete portions | Transfer or process progressively |
| Access model | Usually request a specific portion | Usually consume data as it becomes available |
| Typical purpose | Avoid loading unnecessary data | Avoid accumulating the entire data set |
| Example | Load 100 records at a time | Process incoming chunks continuously |
| Possible tactical fit | Reduce Overhead, Limit Event Response | Reduce Overhead and other resource-related goals |

Both are **approaches**, not automatically tactics in this hierarchy.

---

# 15. Example: Very Large Data Transfer

A poor design might be:

```text
Backend
   ↓
Send Entire Data Set
   ↓
Consumer Stores Everything
   ↓
Consumer Processes Everything
```

Possible consequences:

- Large memory consumption
- Long waiting time
- Large buffers
- Increased serialization cost
- Increased processing demand
- Poor responsiveness

An alternative:

```text
Backend
   │
   ▼
Send Chunk 1
   │
   ▼
Consumer Processes Chunk 1
   │
   ▼
Request / Receive Next Chunk
   │
   ▼
Send Chunk 2
   │
   ▼
Repeat
```

This design can combine several ideas:

```text
Performance
│
└── Control Resource Demand
    │
    ├── Reduce Overhead
    │      └── Chunking / Streaming
    │
    └── Limit Event Response
           └── Request only the next portion when needed
```

The exact classification depends on why the architecture uses chunking or streaming.

---

# 16. Can the Same Approach Support Multiple Tactics?

**Yes.**

An approach should not be permanently assigned to only one tactic.

The correct question is:

> What architectural problem is this approach solving in this context?

## Example: Batching

### Under Reduce Overhead

```text
100 individual operations
        ↓
Batch
        ↓
One combined operation
```

The purpose is to reduce repeated setup or communication overhead.

### Under Limit Event Response

```text
100 incoming events
        ↓
Batch / Coalesce
        ↓
One consolidated response
```

The same approach supports a different tactic.

## Example: Caching

```text
Repeated Computation
        ↓
Caching
        ↓
Reuse Result
```

Here caching supports **Reduce Overhead** because it avoids repeated work.

## Example: Paging

```text
Large Data Set
       ↓
Retrieve Only Required Portion
```

This can support **Reduce Overhead**.

```text
User Action
     ↓
Do Not Load Everything
     ↓
Load Only What Is Needed
```

This can support **Limit Event Response**.

---

# 17. Classification Rule

A useful rule is:

> **Do not classify an approach only by its name. Classify it according to the tactic it supports in the specific architectural context.**

The hierarchy is:

```text
Performance
    ↓
Control Resource Demand
    ↓
Tactic
    ↓
Why is this tactic needed?
    ↓
Possible Approach
    ↓
How does the approach help?
    ↓
Concrete Implementation
```

Example:

```text
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Caching
    ↓
Avoid repeated computation
    ↓
Technology-specific cache implementation
```

---

# 18. Summary

## Core idea

> Performance is not improved only by adding more resources. Another important strategy is to control the demand placed on existing resources.

If demand is uncontrolled:

```text
More Work
    ↓
Resource Saturation
    ↓
Queue Growth
    ↓
Increased Waiting
    ↓
Higher Response Time
    ↓
Timeouts / Failures
    ↓
Potential Instability
```

Therefore:

```text
Control Resource Demand
        ↓
Reduce or control unnecessary work
        ↓
Protect limited resources
        ↓
Reduce waiting and contention
        ↓
Maintain more predictable performance
```

## Final Structure

```text
QUALITY ATTRIBUTE
│
└── Performance
    │
    └── TACTIC CATEGORY
        │
        └── Control Resource Demand
            │
            ├── Manage Sampling Rate
            │      └── Sampling approaches
            │
            ├── Limit Event Response
            │      └── Filtering, coalescing, throttling, batching
            │
            ├── Prioritize Events
            │      └── Priority classification and scheduling approaches
            │
            ├── Reduce Overhead
            │      └── Caching, batching, reuse, deduplication
            │
            ├── Bound Execution Times
            │      └── Time limits, timeouts, cancellation, deadlines
            │
            └── Increase Resource Efficiency
                   └── Efficient processing and resource usage
```

## Important Takeaway

**Tactics describe the architectural action or goal.**

**Approaches describe possible ways of realizing that tactic.**

**Implementations describe the concrete mechanism used in a particular technology or system.**

Example:

```text
Performance
    ↓
Control Resource Demand
    ↓
Reduce Overhead
    ↓
Caching
    ↓
Store a result and reuse it instead of recomputing it
    ↓
Technology-specific cache implementation
```

This separation makes the architectural reasoning technology-independent.
