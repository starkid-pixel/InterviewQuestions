# Baker Hughes — Senior WPF C#/.NET Developer
## Interview Preparation — Question Bank and Answer Guide

## 1. Interview Focus

This JD strongly emphasizes:

- C# / .NET
- WPF and MVVM
- Application architecture
- Threading and concurrency
- Memory and performance
- Responsive UI
- Embedded/hardware integration
- Real-time data
- Custom controls and reusable components
- Debugging across software and hardware
- SOLID and design patterns
- Low-level/object-oriented design

---

# 2. C# / .NET

## Q1. What happens when a C# program runs?

```text
C# source
  ↓
Compiler
  ↓
IL / CIL
  ↓
Assembly
  ↓
CLR
  ↓
JIT
  ↓
Native machine code
```

The CLR provides GC, JIT, exception handling, type safety, threading support, assembly loading, and interoperability.

## Q2. What is CLR?

The Common Language Runtime is the execution environment for managed .NET code. Major responsibilities include garbage collection, JIT compilation, exception handling, type management, threading, assembly loading, and interop.

## Q3. .NET Framework vs modern .NET?

.NET Framework is the older Windows-focused implementation. .NET Core introduced a modern, modular, cross-platform implementation, which evolved into unified modern .NET.

```text
.NET Framework → .NET Core → .NET 5+ → modern .NET
```

## Q4. Class vs struct?

A class is a reference type; a struct is a value type. Do not reduce this to “class = heap, struct = stack.” Storage depends on context. Consider value semantics, copying, mutability, boxing, size, and lifetime.

## Q5. What is boxing?

```csharp
int x = 10;
object o = x;
```

The value type is boxed into an object. Unboxing extracts the value. Excessive boxing can create allocations and affect performance.

## Q6. What is a delegate?

A type-safe reference to a method. Common uses are callbacks, events, and passing behavior.

## Q7. Delegate vs event?

A delegate can generally be invoked by code holding it. An event restricts raising the event to the declaring type and provides a controlled publish/subscribe mechanism.

## Q8. Can events cause memory leaks?

Yes. If a long-lived publisher holds a subscription to a short-lived subscriber, the subscriber remains reachable.

```text
Long-lived publisher
       ↓ event
Short-lived subscriber
```

Use appropriate unsubscription, weak events, or lifetime management.

---

# 3. Memory Management / GC

## Q9. Explain .NET GC.

The GC identifies unreachable managed objects and reclaims their memory.

```text
New object
   ↓
Gen 0
   ↓ survives
Gen 1
   ↓ survives
Gen 2
```

## Q10. Does every GC go Gen 0 → Gen 1 → Gen 2?

No.

```text
Gen 0 collection → Gen 0
Gen 1 collection → Gen 0 + Gen 1
Gen 2 collection → Gen 0 + Gen 1 + Gen 2
```

The runtime chooses the collection scope based on runtime conditions and GC heuristics.

## Q11. When is Gen 2 collected?

The runtime decides based on allocation pressure, memory conditions, thresholds, GC history, and heuristics. Gen 2 collections are generally less frequent and more expensive.

## Q12. What is LOH?

The Large Object Heap handles large allocations and has different allocation/GC characteristics. Large allocations can contribute to memory pressure and fragmentation.

## Q13. Can a managed application have a memory leak?

Yes. An object can be logically unused but still reachable from a GC root.

Common causes:

- Static references
- Events
- Timers
- Caches
- Navigation references
- Long-lived services
- Global collections

## Q14. What are GC roots?

References from which reachability is determined, such as active stack references, static references, and runtime-managed references.

## Q15. IDisposable vs GC?

GC reclaims managed memory nondeterministically. IDisposable provides deterministic cleanup for resources such as file handles, native resources, device handles, and network resources.

`using` calls `Dispose`; it does not force GC.

---

# 4. Tasks, async/await, Threads

## Q16. Does async create a thread?

No. async/await primarily provides asynchronous control flow. I/O can proceed without blocking the calling thread. CPU-bound work may require background execution.

## Q17. CPU-bound vs I/O-bound?

CPU-bound work consumes processor time, such as image processing or calculations.

I/O-bound work waits for external operations, such as device, network, database, or file I/O.

Use asynchronous APIs for I/O. Move CPU-heavy work off the UI thread when appropriate.

## Q18. What happens at await?

Conceptually:

```text
Method starts
  ↓
Operation starts
  ↓
await incomplete operation
  ↓
Control returns
  ↓
Operation completes
  ↓
Continuation resumes
```

Continuation behavior depends on the synchronization context/execution environment.

## Q19. Why can .Result/.Wait() be dangerous in WPF?

They synchronously block the UI thread. If the awaited operation needs to resume through the UI synchronization context, this can produce blocking or deadlock.

Prefer:

```csharp
var result = await service.GetDataAsync();
```

## Q20. Thread vs Task?

A Thread represents an execution thread. A Task represents an asynchronous unit of work and is a higher-level abstraction that composes with async/await.

## Q21. What is ThreadPool?

A pool of reusable worker threads used for background work. Tasks commonly use ThreadPool threads for CPU-bound execution.

## Q22. What is a race condition?

Concurrent operations access shared state and the result depends on timing.

Solutions can include:

- Locking
- Atomic operations
- Immutable data
- Concurrent collections
- Message passing
- Ownership

## Q23. What is deadlock?

Two or more threads wait indefinitely for resources held by each other.

```text
Thread A: Lock 1 → waits for Lock 2
Thread B: Lock 2 → waits for Lock 1
```

Avoid inconsistent lock ordering, excessive locking, and unnecessary synchronization.

## Q24. lock vs SemaphoreSlim?

`lock` provides mutual exclusion for synchronous critical sections. `SemaphoreSlim` can limit concurrency and can be awaited asynchronously.

---

# 5. Responsive WPF UI

## Q25. How would you design a responsive WPF UI?

Core principle:

> Keep the UI thread free for input, layout, rendering, and UI work.

Typical architecture:

```text
Hardware / Network
       ↓
Communication Layer
       ↓
Background Processing
       ↓
Buffer / Queue
       ↓
Processing / Aggregation
       ↓
ViewModel
       ↓
UI Thread
       ↓
WPF
```

Use async I/O, background CPU processing, buffering, batching, throttling, virtualization, and controlled UI update rates.

## Q26. The UI freezes for five seconds. How do you investigate?

Do not immediately prescribe async.

Investigate:

```text
UI freeze
  ↓
UI thread blocked?
  ├─ CPU-bound work
  ├─ synchronous I/O
  ├─ lock contention
  ├─ deadlock
  ├─ GC
  ├─ layout
  ├─ rendering
  ├─ binding
  └─ large collection
```

Use profiling, tracing, dumps, and measurements to find the actual cause.

## Q27. Why shouldn't hardware communication happen on the UI thread?

Device communication can block or have unpredictable latency. Blocking the UI thread delays input, layout, and rendering.

Use a communication service with asynchronous/background execution.

## Q28. Device sends 10,000 messages/sec. How do you display them?

Do not update the UI for every message.

```text
Device
  ↓
Producer
  ↓
Buffer / Channel / Queue
  ↓
Consumer
  ↓
Processing
  ↓
Aggregation / Sampling
  ↓
Controlled UI updates
```

The device rate should not dictate the UI rendering rate.

## Q29. Why can ObservableCollection become a performance problem?

Each modification can generate a collection-change notification. Thousands of individual additions can therefore cause excessive UI work.

Consider batching, range updates, virtualization, data virtualization, and throttling.

## Q30. Should Dispatcher be used for every background operation?

No. Dispatcher is for UI-affine work. Flooding it with thousands of updates can itself make the UI unresponsive.

---

# 6. WPF / MVVM

## Q31. Explain MVVM.

```text
View
 ↓
ViewModel
 ↓
Application / Domain Services
 ↓
Infrastructure
```

The View handles presentation. The ViewModel exposes presentation state and commands. Business/application logic should not be embedded directly in the View.

## Q32. Where should hardware communication live?

Prefer:

```text
View
 ↓
ViewModel
 ↓
Application Service
 ↓
Device Abstraction
 ↓
Hardware Adapter
 ↓
Device
```

The ViewModel should not directly open serial ports or call vendor-specific SDKs.

## Q33. What is DataContext?

It is the default source for WPF bindings. It is inherited through the element tree unless overridden.

## Q34. What is INotifyPropertyChanged?

It lets an object notify the binding system that a property changed.

```csharp
PropertyChanged?.Invoke(
    this,
    new PropertyChangedEventArgs(nameof(Name)));
```

## Q35. ObservableCollection vs INotifyPropertyChanged?

`INotifyPropertyChanged` communicates property changes.

`ObservableCollection<T>` communicates collection changes such as add, remove, move, and replace.

## Q36. What is DependencyProperty?

WPF's property system supports binding, styles, animation, inheritance, metadata, defaults, and property precedence.

## Q37. DependencyProperty vs INotifyPropertyChanged?

DependencyProperty belongs to the WPF property system and is heavily used by WPF controls. INotifyPropertyChanged is a notification contract commonly used by ViewModels.

## Q38. UserControl vs CustomControl?

Use UserControl for composition of existing controls. Use CustomControl when creating a reusable control whose appearance should be defined through styles/templates.

## Q39. Why use ControlTemplate?

It allows the visual structure of a control to be replaced without changing its core behavior.

## Q40. What are WPF triggers?

They change presentation behavior when specified property/event conditions occur.

## Q41. StaticResource vs DynamicResource?

StaticResource resolves resource lookup during initialization/resource resolution. DynamicResource maintains a runtime resource reference and supports resource changes at runtime.

## Q42. Logical Tree vs Visual Tree?

The logical tree represents logical UI structure. The visual tree represents actual rendered elements, including template-generated visuals.

Important for resource lookup, event routing, rendering, and traversal.

---

# 7. WPF Memory Leaks

## Q43. Can WPF leak memory despite GC?

Yes.

Common causes:

- Event subscriptions
- Static references
- Timers
- DispatcherTimer
- Event aggregators
- Caches
- Navigation history
- Long-lived services

## Q44. ViewModel remains after its Window closes. How investigate?

Find the retention path:

```text
ViewModel
  ↓
Who references it?
  ↓
Who references that object?
  ↓
...
  ↓
GC root
```

Use a memory profiler/dump to identify the root.

---

# 8. WPF Performance

## Q45. How do you optimize a WPF DataGrid with 100,000 records?

Consider:

- UI virtualization
- Data virtualization
- Paging
- Incremental loading
- Efficient binding
- Reduced notifications
- Reduced visual tree complexity
- Background data retrieval

## Q46. What can make WPF rendering slow?

Potential causes include:

- Large visual tree
- Expensive layout
- Excessive bindings
- Frequent property changes
- Excessive collection changes
- Heavy templates
- Rendering-heavy visuals
- Too much work on UI thread

---

# 9. SOLID / Design Principles

## Q47. SRP?

A class should have a focused responsibility and coherent reason to change.

SRP does not mean “one method per class.”

## Q48. OCP?

Design stable parts so new variations can be added with minimal modification to existing behavior.

Ask:

> What is expected to vary?

## Q49. LSP?

Subtypes must remain behaviorally substitutable for the abstraction they implement.

## Q50. ISP?

Clients should not depend on methods they do not need. Prefer focused interfaces.

## Q51. DIP?

High-level policy should not directly depend on low-level implementation details. Depend on suitable abstractions.

---

# 10. Design Patterns

## Q52. Strategy?

Use when an algorithm/behavior is a meaningful axis of variation.

```text
IDataProcessingStrategy
 ├── StrategyA
 ├── StrategyB
 └── StrategyC
```

## Q53. State?

Use when behavior changes substantially according to object state.

```text
Disconnected
 ↓
Connecting
 ↓
Connected
 ↓
Running
 ↓
Error
 ↓
Recovering
```

## Q54. Adapter?

Use when an existing interface does not match the interface expected by your application.

```text
Application
 ↓
IDevice
 ↓
Vendor A Adapter
Vendor B Adapter
Vendor C Adapter
```

## Q55. Command?

Encapsulates a request as an object. In WPF, ICommand represents UI actions and supports CanExecute.

## Q56. Factory Method?

Encapsulates object creation when the concrete type is a variation point.

## Q57. Abstract Factory?

Creates families of related objects.

## Q58. Template Method?

Defines an algorithm skeleton in a base abstraction and lets subclasses customize selected steps.

---

# 11. Low-Level Design

## Q59. Parking Lot

Possible model:

```text
ParkingLot
 ↓
ParkingFloor
 ↓
ParkingSpot
 ↓
Vehicle
```

Possible variation:

```text
ParkingStrategy
 ├── Nearest
 ├── Cheapest
 └── EVFirst
```

Staff-level follow-up: multiple entrances concurrently allocating the same spot. Discuss atomicity, synchronization, or transaction boundaries depending on deployment.

## Q60. Vending Machine

```text
VendingMachine
 ├── Inventory
 ├── Payment
 ├── Product
 └── State
```

States:

```text
Idle
 ↓
ProductSelected
 ↓
PaymentPending
 ↓
PaymentReceived
 ↓
Dispensing
 ↓
Idle
```

Payment can be Strategy; lifecycle behavior can be State.

## Q61. Chess

```text
ChessGame
 ├── Board
 ├── Player
 ├── Move
 └── RulesEngine
```

Rules can be separated:

```text
MovementRules
CheckRule
CheckmateRule
CastlingRule
EnPassantRule
PromotionRule
```

Piece movement can use strategies.

The key question is:

> What varies, and where should that variation live?

---

# 12. Hardware Integration

## Q62. How would you architect hardware communication?

```text
Application
 ↓
IDevice
 ↓
Device Service
 ↓
Transport / Adapter
 ↓
Serial / USB / TCP
 ↓
Hardware
```

Keep protocol/vendor-specific details out of application logic.

## Q63. Multiple hardware vendors?

Use a stable abstraction and adapters:

```text
IDevice
 ├── VendorAAdapter
 ├── VendorBAdapter
 └── VendorCAdapter
```

## Q64. Device disconnects unexpectedly?

Use explicit state management:

```text
Connected
 ↓
Disconnected
 ↓
Reconnecting
 ↓
Connected
```

Consider timeouts, cancellation, retry/backoff, buffer behavior, error reporting, logging, and UI state.

## Q65. How would you send commands to hardware?

Separate command creation from transport:

```text
View
 ↓
ViewModel
 ↓
Application Service
 ↓
Device Command
 ↓
Device Service
 ↓
Transport
 ↓
Hardware
```

Use async operations and cancellation when appropriate.

---

# 13. Real-Time Data

## Q66. Design a real-time data pipeline.

```text
Device
 ↓
Producer
 ↓
Bounded Buffer
 ↓
Processing
 ↓
Aggregation
 ↓
UI Update Scheduler
 ↓
ViewModel
 ↓
WPF
```

## Q67. What is backpressure?

When the producer is faster than the consumer.

```text
Producer: 10,000/sec
Consumer: 1,000/sec
```

A queue can grow without bound unless a policy exists.

Possible policies:

- Bounded queue
- Drop obsolete data
- Sampling
- Aggregation
- Throttling
- Producer rate control

The correct policy depends on whether every event is required.

## Q68. Should real-time data ever be dropped?

It depends on requirements.

Telemetry visualization may drop obsolete intermediate values. Commands, safety events, financial transactions, and audit data may require every event.

---

# 14. C++ / .NET Interop

## Q69. How can C# communicate with C++?

Possible approaches:

```text
C#
 ├── P/Invoke → native C API
 ├── C++/CLI → C++ wrapper
 └── COM / other interop
```

Choice depends on the existing native API, ownership, complexity, deployment, ABI, and performance requirements.

## Q70. What should you worry about at a managed/native boundary?

- Ownership
- Lifetime
- Memory allocation/deallocation
- ABI compatibility
- Marshaling
- Threading requirements
- Exceptions
- Native handles
- Disposal
- Crash diagnostics

---

# 15. Architecture Scenarios

## Q71. Design a large WPF application with 100+ screens.

Consider modularization:

```text
Shell
 ├── Module A
 ├── Module B
 ├── Module C
 └── Module D
```

Keep module contracts explicit and prevent shared infrastructure from becoming a dumping ground.

## Q72. How do you prevent modules becoming tightly coupled?

Use:

- Stable abstractions
- Dependency inversion
- Clear module contracts
- Explicit ownership
- Appropriate application/domain events
- Avoid unnecessary direct module references

## Q73. How would you modernize a legacy WPF application?

Prefer incremental modernization when practical:

```text
Legacy
 ↓
Dependency analysis
 ↓
Architectural boundaries
 ↓
Extract business logic
 ↓
Introduce abstractions
 ↓
DI
 ↓
MVVM where useful
 ↓
Modularize
 ↓
Incrementally replace areas
```

Do not assume a rewrite is automatically the best option.

## Q74. How would you make WPF testable?

Separate:

```text
View
 ↓
ViewModel
 ↓
Application Services
 ↓
Interfaces
 ↓
Infrastructure
```

Inject dependencies. Keep business/application logic independent of actual WPF controls and physical hardware.

---

# 16. Staff-Level Troubleshooting

## Q75. UI freezes intermittently.

Approach:

1. Reproduce if possible.
2. Determine whether UI thread is blocked.
3. Capture diagnostics during the freeze.
4. Check CPU, I/O, locking, GC, layout, rendering, and binding.
5. Identify root cause.
6. Fix.
7. Measure before/after.

## Q76. Memory increases after opening and closing screens.

Investigate:

- Events
- Static references
- Timers
- Navigation
- Caches
- Event aggregators
- Long-lived services
- ViewModel references

Find the retention path to a GC root.

## Q77. Device produces data faster than UI consumes.

Decouple producer and consumer:

```text
Hardware
 ↓
Producer
 ↓
Bounded Queue
 ↓
Processing
 ↓
Aggregation/Sampling
 ↓
UI
```

Define a data-loss policy from requirements.

## Q78. Third-party hardware SDK randomly crashes.

Isolate it behind an infrastructure boundary and investigate:

- Native/managed boundary
- Lifetime
- Threading requirements
- ABI/API
- SDK version
- Crash dumps
- Reproducibility
- Vendor documentation

---

# 17. Questions That Test Staff-Level Judgment

Be prepared for:

- Why did you choose this pattern?
- Why not inheritance?
- Why not simply use interfaces?
- Why Strategy instead of if/else?
- Why introduce another abstraction?
- Isn't this over-engineering?
- What happens if requirements change?
- What is the performance cost?
- What happens under high load?
- What happens if the device disconnects?
- How would you test it?
- How would you monitor it?
- What are the trade-offs?
- What would you deliberately NOT abstract?

The strongest answers discuss trade-offs rather than claiming one design is universally correct.

---

# 18. Highest-Priority 35 Questions

If preparation time is limited, prioritize:

1. How do you design a responsive WPF UI?
2. Why can WPF UI freeze?
3. Device sends 10,000 messages/sec — design the solution.
4. Producer/consumer design.
5. Backpressure.
6. async/await.
7. Does async create a thread?
8. Why is `.Result` dangerous in WPF?
9. GC generations.
10. Managed memory leaks.
11. WPF memory leaks.
12. How to diagnose a memory leak.
13. MVVM.
14. Where hardware communication belongs in MVVM.
15. DataContext.
16. INotifyPropertyChanged.
17. ObservableCollection.
18. DependencyProperty.
19. UserControl vs CustomControl.
20. WPF performance/DataGrid.
21. Modular WPF architecture.
22. Hardware abstraction.
23. Multiple device vendors.
24. Strategy.
25. State.
26. Adapter.
27. Command.
28. Factory.
29. Parking Lot.
30. Vending Machine.
31. Chess.
32. Legacy WPF modernization.
33. Testable WPF architecture.
34. UI-freeze diagnosis.
35. Device disconnect/reconnection.

---

# 19. The Core Design You Should Be Able to Whiteboard

For this JD, be prepared to design:

```text
                         WPF
                          │
                         MVVM
                          │
                      ViewModel
                          │
                  Application Layer
                          │
                ┌─────────┴─────────┐
                │                   │
          Command Path         Data Path
                │                   │
                ▼                   ▼
         Device Service        Data Pipeline
                │                   │
                ▼              Buffer / Queue
          Device Adapter             │
                │                Processing
        ┌───────┼───────┐            │
        ↓       ↓       ↓       Aggregate/Sample
      Serial   USB     TCP           │
        │       │       │            ▼
        └───────┼───────┘        ViewModel
                │                   │
                ▼                   ▼
             Hardware             WPF UI
```

Cross-cutting concerns:

```text
Threading
Memory
Performance
Error Handling
Logging
Diagnostics
Cancellation
Testing
```

---

# 20. Interview Answer Framework

For architecture/scenario questions, use this sequence:

```text
1. Clarify requirements
        ↓
2. Identify architectural drivers
        ↓
3. Identify responsibilities
        ↓
4. Identify what varies
        ↓
5. Define boundaries
        ↓
6. Define abstractions
        ↓
7. Choose patterns where justified
        ↓
8. Discuss concurrency/performance
        ↓
9. Discuss failure scenarios
        ↓
10. Discuss testing
        ↓
11. Explain trade-offs
```

Do not begin with:

> "I'll use Strategy."

Begin with:

> "What is the requirement, what is expected to vary, and what needs to remain stable?"

Patterns should emerge from the problem.

---

# 21. Final Mental Model

For this role, connect the topics rather than studying them independently:

```text
C# / CLR
    ↓
Memory + GC
    ↓
Tasks + async/await
    ↓
Threading + synchronization
    ↓
WPF UI Thread
    ↓
MVVM + Binding
    ↓
Real-time Data Pipeline
    ↓
Hardware Abstraction
    ↓
Performance / Responsiveness
    ↓
Architecture
```

The strongest Staff-level answer is not:

> "I know WPF, C#, and design patterns."

It is:

> "I can identify the architectural problem, isolate responsibilities, keep the UI responsive, handle concurrency and hardware failures, control high-frequency data, make the system testable, and explain the trade-offs behind the design."
