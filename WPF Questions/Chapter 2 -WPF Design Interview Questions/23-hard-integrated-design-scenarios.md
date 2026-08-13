# Chapter 23 — Hard Integrated Design Scenarios

These questions combine several WPF concepts and are excellent senior/architect interview exercises.

# Scenario 1 — Enterprise WPF Application

## Requirement

Design a desktop application with:
- 100+ screens
- REST APIs
- authentication
- role-based authorization
- navigation
- dialogs
- background processing
- logging
- caching
- offline mode
- multiple teams

## Answer

```text
                         Shell
                           |
                     Navigation
                           |
          +----------------+----------------+
          |                |                |
       Module A         Module B         Module C
          |                |                |
      ViewModels       ViewModels       ViewModels
          |                |                |
      App Services     App Services     App Services
          +----------------+----------------+
                           |
                      Shared Contracts
                           |
        +------------------+------------------+
        |                  |                  |
       API              Local DB          Auth
        |                  |                  |
     Backend           Offline Store     Identity
```

Use modular boundaries, dependency injection, interfaces, centralized navigation/dialog services, and explicit application lifecycle management.

# Scenario 2 — One Million Records

## Requirement

Users can search, sort, filter, select, and export millions of records.

## Answer

Do not load millions of objects.

```text
Search UI
   |
Debounce
   |
ViewModel
   |
Search Service
   |
API
   |
Database
```

Server handles:
- filtering
- sorting
- paging
- aggregation

Client handles:
- current page
- selection state
- UI virtualization
- cancellation
- presentation

# Scenario 3 — 100,000 CPU-heavy jobs

## Answer

Use bounded concurrency.

```text
User
 |
AsyncCommand
 |
Coordinator
 |
Bounded Workers
 |
CPU processing
 |
Progress
 |
Cancellation
```

Do not launch 100,000 uncontrolled tasks.

Example:

```csharp
await Parallel.ForEachAsync(
    items,
    new ParallelOptions
    {
        MaxDegreeOfParallelism =
            Environment.ProcessorCount,
        CancellationToken = token
    },
    async (item, ct) =>
    {
        await ProcessAsync(item, ct);
    });
```

# Scenario 4 — Real-time dashboard

Requirements:
- 10,000 updates/second
- DataGrid
- live charts
- no UI freeze

## Answer

```text
WebSocket
   |
Background receiver
   |
Bounded Channel
   |
Batch/coalesce
   |
UI Dispatcher
   |
Virtualized DataGrid
```

Do not dispatch every message individually.

# Scenario 5 — Offline-first application

Use:

```text
Local database
     |
Local repository
     |
Application service
     |
ViewModel
```

Synchronize pending changes when online.

Track:

```text
Pending
Synced
Failed
Conflict
```

Define conflict rules explicitly.

# Scenario 6 — Memory leak

Requirement:

> Every time a customer window opens and closes, memory grows.

## Investigation

1. Reproduce repeatedly.
2. Force/observe GC.
3. Take heap snapshots.
4. Compare Window/ViewModel instance counts.
5. Inspect retention paths.
6. Look for:
   - static events
   - event aggregators
   - timers
   - Dispatcher callbacks
   - long-lived services
   - global collections
   - event subscriptions

## Design fix

Give short-lived components an explicit lifecycle:

```csharp
public interface IDisposableViewModel : IDisposable
{
}
```

Or use an async lifecycle:

```csharp
public interface IAsyncDisposableViewModel
{
    ValueTask DisposeAsync();
}
```

# Scenario 7 — Dialog architecture

Requirement:

> 50 ViewModels need dialogs, but none may reference WPF Window classes.

Use:

```text
ViewModel
 |
IDialogService
 |
WpfDialogService
 |
Window
```

Unit tests use:

```text
FakeDialogService
```

# Scenario 8 — Navigation memory management

If navigation history stores complete ViewModels:

```text
History
 |
VM1
 |
large object graph
```

memory can grow.

Alternative:

```csharp
public record NavigationEntry(
    Type ViewModelType,
    object? Parameter);
```

Recreate the ViewModel when going back if state preservation is not required.

# Scenario 9 — API failure

A robust flow:

```text
API
 |
HTTP failure
 |
API client
 |
Classify failure
 |
Application service
 |
ViewModel state
 |
User-friendly message
```

Do not show raw exception messages to users.

# Scenario 10 — Senior follow-up questions

After you present any design, expect:

### Why?

Explain the trade-off.

### What if the network fails?

Explain cancellation, retry, timeout, and offline behavior.

### What if the operation takes 30 seconds?

Explain asynchronous execution and progress.

### What if the user navigates away?

Cancel or detach work according to lifecycle requirements.

### What if memory keeps increasing?

Explain ownership, event subscriptions, timers, caches, and profiling.

### How do you test it?

Explain interfaces, dependency injection, fakes, and unit tests.

### How does it scale?

Discuss data volume, concurrency, UI virtualization, batching, caching, and module boundaries.

### What would you change for 10 teams?

Introduce module contracts, ownership boundaries, shared infrastructure, CI checks, and architectural governance.

# Interview Answer Framework

For almost every WPF design question, answer in this order:

1. Clarify requirements.
2. Identify major components.
3. Draw the architecture.
4. Define responsibilities.
5. Define communication.
6. Explain threading.
7. Explain lifecycle.
8. Explain error handling.
9. Explain performance.
10. Explain testability.
11. Discuss trade-offs.
12. Address failure scenarios.

That structure makes the answer sound architectural rather than simply API-focused.
