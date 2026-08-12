# C# Memory Management --- Tough Interview Questions & Answers

## 1. GC and Object Lifetime

### Q1. If an object is allocated on the managed heap, when exactly does it become eligible for garbage collection?

**Answer:**

An object becomes eligible for garbage collection when it is no longer
reachable from any **GC root**.

GC does not determine whether an object is still logically needed by the
business. It determines whether the object is reachable.

``` csharp
void Process()
{
    Customer customer = new Customer();
    customer.Process();
}
```

After `Process()` finishes, assuming there are no other references to
`customer`, the object can become unreachable and therefore eligible for
GC.

However:

> **Eligible for GC does not mean immediately collected.**

The runtime decides when to perform a collection.

The conceptual sequence is:

``` text
Object becomes unreachable
        ↓
Object becomes eligible for GC
        ↓
GC performs a collection
        ↓
Memory is reclaimed
```

------------------------------------------------------------------------

### Q2. Can an object be collected while a method is still executing?

**Answer:**

Yes, potentially.

The important concept is that **source-code scope is not the same as
runtime liveness**.

The JIT can determine that a local reference is no longer needed after
its last useful use. Therefore, an object can potentially become
unreachable before the method has returned.

``` csharp
void Process()
{
    Customer customer = new Customer();

    customer.Process();

    DoSomethingElse();
}
```

After the last use of `customer`, the JIT may determine that the
reference is dead.

A strong interview answer is:

> The lifetime of an object is based on reachability, and the
> JIT/runtime can determine when references are no longer live.
> Therefore, a variable remaining syntactically in scope does not
> necessarily mean the object must remain reachable.

------------------------------------------------------------------------

### Q3. What are GC roots?

**Answer:**

GC roots are references from which the garbage collector begins
determining object reachability.

Common sources include:

-   Live references on thread stacks
-   Static references
-   Runtime handles
-   References associated with active execution
-   Objects involved in finalization and other runtime mechanisms

For example:

``` text
GC Root
   ↓
Customer
   ↓
Address
```

`Customer` and `Address` are reachable and therefore considered live.

If the root no longer references `Customer`, and no other root can reach
it, the object graph may become eligible for collection.

------------------------------------------------------------------------

### Q4. Does this immediately destroy the object?

``` csharp
var obj = new MyObject();
obj = null;
```

**Answer:**

No.

Setting `obj` to `null` only removes that particular reference.

If no other references exist, the object may become eligible for GC.

It is not immediately destroyed.

``` text
Reference removed
       ↓
Object may become unreachable
       ↓
Object becomes eligible for GC
       ↓
Future GC collection
```

------------------------------------------------------------------------

### Q5. Can an object be eligible for GC even though its variable is technically still in scope?

**Answer:**

Yes.

Scope is a language-level concept. Object liveness is a runtime concept.

The JIT may determine that a local variable is no longer needed after
its last use.

Therefore:

> **A variable being in lexical scope does not guarantee that its object
> remains reachable.**

------------------------------------------------------------------------

### Q6. Does going out of scope always make an object eligible for GC?

**Answer:**

No.

Suppose:

``` csharp
void Process()
{
    Customer customer = new Customer();
    SomeOtherObject.Customer = customer;
}
```

After `Process()` returns, the local variable disappears, but:

``` text
SomeOtherObject
      ↓
   Customer
```

may still keep the object alive.

Therefore:

> **Scope and object lifetime are related, but they are not the same
> thing.**

------------------------------------------------------------------------

## 2. Generational GC

### Q7. Why does .NET use generational garbage collection?

**Answer:**

Most objects have short lifetimes.

For example:

``` text
Temporary request object
Temporary string
Temporary DTO
Temporary collection
```

Instead of repeatedly scanning the entire heap, the GC divides objects
into generations and focuses collection effort on younger generations
where garbage is more likely to exist.

The main generations are:

-   Gen 0
-   Gen 1
-   Gen 2

This improves GC efficiency.

------------------------------------------------------------------------

### Q8. What are Gen 0, Gen 1, and Gen 2?

**Answer:**

**Gen 0**

Contains newly allocated objects. It is collected frequently and is
usually the cheapest collection.

**Gen 1**

Acts as a buffer between short-lived and long-lived objects.

**Gen 2**

Contains objects that have survived enough collections to be considered
long-lived.

Examples of long-lived objects can include:

-   Application-wide caches
-   Static objects
-   Long-lived services
-   Configuration objects

------------------------------------------------------------------------

### Q9. Why is a Gen 2 collection generally more expensive than a Gen 0 collection?

**Answer:**

Gen 2 contains long-lived objects, so there can be significantly more
memory to examine.

A Gen 2 collection can involve a much larger portion of the managed
heap.

Therefore:

``` text
Gen 0 collection → usually cheaper
Gen 1 collection → more expensive
Gen 2 collection → potentially much more expensive
```

Frequent Gen 2 collections can be an important performance warning.

------------------------------------------------------------------------

### Q10. If an object survives a Gen 0 collection, what happens?

**Answer:**

It may be promoted to a higher generation.

The general lifecycle is:

``` text
New object
   ↓
Gen 0
   ↓ survives
Gen 1
   ↓ survives
Gen 2
```

Promotion is one of the mechanisms used by the generational GC to
distinguish short-lived and long-lived objects.

------------------------------------------------------------------------

### Q11. Can a Gen 2 object reference a Gen 0 object?

**Answer:**

Yes.

The GC must be able to track such references efficiently.

.NET uses runtime mechanisms such as **write barriers** and
remembered-set/card-table techniques to track relevant references
between generations.

This prevents the GC from having to blindly scan the entire older
generation whenever it collects Gen 0.

------------------------------------------------------------------------

### Q12. What is a write barrier?

**Answer:**

A write barrier is runtime machinery used to track certain reference
assignments, particularly references that can affect generational GC.

For example, if an older object is modified to reference a younger
object, the GC needs to know about that relationship when collecting the
younger generation.

The write barrier helps maintain this information efficiently.

------------------------------------------------------------------------

## 3. Large Object Heap

### Q13. What is the Large Object Heap (LOH)?

**Answer:**

The Large Object Heap is a special managed heap area used for
sufficiently large allocations.

Large objects are treated differently because moving large blocks of
memory can be expensive.

Typical examples include:

``` csharp
byte[] buffer = new byte[largeSize];
```

Large arrays are a common source of LOH allocations.

------------------------------------------------------------------------

### Q14. Why are large objects treated differently?

**Answer:**

Large objects are expensive to copy and move.

A normal compacting collection can move objects to reduce fragmentation,
but repeatedly moving very large objects would be expensive.

Therefore, the runtime treats the LOH differently from the normal
small-object heaps.

------------------------------------------------------------------------

### Q15. Why can repeatedly allocating and releasing large byte arrays still cause high memory usage?

**Answer:**

Several factors can contribute:

-   LOH allocations are expensive
-   Objects may not be immediately collected
-   The heap may retain committed memory for future allocations
-   Fragmentation can occur
-   Some objects may still be reachable

Therefore, observing high process memory does not automatically mean
that the GC is failing.

------------------------------------------------------------------------

### Q16. Does the GC immediately return freed managed memory to Windows?

**Answer:**

No.

This is an important distinction.

``` text
Managed memory no longer occupied by live objects
                 ≠
Process working set immediately decreases
```

The runtime may retain memory for future managed allocations.

Therefore, a process can continue to show substantial memory usage even
after garbage has been collected.

------------------------------------------------------------------------

### Q17. What is LOH fragmentation?

**Answer:**

LOH fragmentation occurs when free regions become distributed among
allocated regions, making it harder to satisfy large contiguous
allocations efficiently.

A workload involving many different-sized large allocations can
contribute to fragmentation.

In appropriate scenarios, LOH compaction can be requested.

------------------------------------------------------------------------

## 4. Managed Memory Leaks

### Q18. Can a garbage-collected language have memory leaks?

**Answer:**

Yes.

GC removes objects that are **unreachable**.

It does not remove objects that are still strongly reachable, even if
the application no longer logically needs them.

Example:

``` csharp
private readonly List<byte[]> _cache = new();

public void Process()
{
    _cache.Add(new byte[10_000_000]);
}
```

If `Process()` is called repeatedly, the list maintains references to
all the arrays.

``` text
Service
   ↓
_cache
   ↓
List<byte[]>
   ↓
byte[]
byte[]
byte[]
...
```

The objects remain reachable, so GC cannot collect them.

This is commonly described as a **managed memory leak** or **memory
retention problem**.

------------------------------------------------------------------------

### Q19. How can an event cause a memory leak?

**Answer:**

Consider a long-lived publisher and a short-lived subscriber:

``` csharp
publisher.SomethingHappened += subscriber.Handle;
```

The publisher's event can maintain a reference to the subscriber's event
handler.

If the publisher remains alive for a long time, the subscriber may also
remain reachable.

Conceptually:

``` text
Long-lived Publisher
        ↓
      Event
        ↓
    Subscriber
```

If the subscriber is supposed to have a shorter lifetime, the
subscription can unintentionally extend its lifetime.

Unsubscribing is one common solution:

``` csharp
publisher.SomethingHappened -= subscriber.Handle;
```

------------------------------------------------------------------------

### Q20. What is the key difference between a memory leak in C++ and a managed memory leak in C#?

**Answer:**

In C++, a common leak occurs when dynamically allocated memory is no
longer referenced and the program fails to explicitly release it.

In .NET, unreachable managed objects are normally reclaimed
automatically.

However, managed objects can remain reachable through unintended
references such as:

-   Static fields
-   Long-lived caches
-   Event subscriptions
-   Timers
-   Collections
-   Closures
-   Long-lived services

Therefore:

> **In .NET, memory retention caused by unintended reachability is a
> major form of memory leak.**

------------------------------------------------------------------------

## 5. IDisposable and Resource Management

### Q21. Why do we need IDisposable if .NET has garbage collection?

**Answer:**

GC manages **managed memory**.

It does not provide deterministic release of external resources such as:

-   File handles
-   Database connections
-   Network sockets
-   OS handles
-   Native resources

`IDisposable` provides a deterministic mechanism for releasing such
resources.

Example:

``` csharp
using var stream = File.OpenRead("data.txt");
```

The stream is disposed when the scope is exited.

------------------------------------------------------------------------

### Q22. Does calling Dispose() free managed memory?

**Answer:**

Not directly.

`Dispose()` is a convention for releasing resources and performing
deterministic cleanup.

It does not mean:

> "Tell the GC to immediately destroy this object."

The object itself can still exist after `Dispose()` if references to it
remain.

------------------------------------------------------------------------

### Q23. What happens if you don't dispose a FileStream?

**Answer:**

The underlying operating-system resources may remain held longer than
intended.

This can lead to:

-   File locks
-   Resource exhaustion
-   Too many open handles
-   Poor application behavior

Finalization may eventually provide cleanup for some resource-owning
types, but relying on it is not a substitute for deterministic disposal.

------------------------------------------------------------------------

### Q24. Does GC call Dispose() automatically?

**Answer:**

No.

GC does not automatically invoke `Dispose()` simply because a type
implements `IDisposable`.

You should use deterministic disposal:

``` csharp
using var resource = new Resource();
```

or explicitly:

``` csharp
resource.Dispose();
```

------------------------------------------------------------------------

### Q25. What is the relationship between IDisposable and finalizers?

**Answer:**

They solve different aspects of resource management.

`Dispose()` provides **deterministic cleanup**.

A finalizer provides a runtime fallback for certain unmanaged-resource
scenarios.

A common pattern is:

``` text
Dispose()
   ↓
Release resources deterministically

Finalizer
   ↓
Fallback cleanup if appropriate
```

Modern .NET code should generally prefer `SafeHandle` for unmanaged
handles rather than implementing raw finalizers unnecessarily.

------------------------------------------------------------------------

## 6. Finalization

### Q26. What happens when a class has a finalizer?

Example:

``` csharp
class MyClass
{
    ~MyClass()
    {
        // cleanup
    }
}
```

**Answer:**

A finalizable object requires special treatment by the GC.

It is associated with the finalization mechanism and generally survives
at least one additional GC phase before its memory can be reclaimed.

Therefore finalizers introduce additional GC overhead.

------------------------------------------------------------------------

### Q27. Why can finalizers hurt performance?

**Answer:**

Finalizable objects cannot simply be reclaimed during the first normal
discovery of garbage.

They must go through the finalization process.

This can:

-   Increase object lifetime
-   Increase GC work
-   Delay memory reclamation
-   Add finalizer-thread processing

Therefore:

> **Do not add finalizers unless there is a genuine unmanaged-resource
> requirement.**

------------------------------------------------------------------------

### Q28. When should you implement a finalizer?

**Answer:**

Only when the type directly owns unmanaged resources that require
finalization as a safety net.

For most resource-management scenarios, prefer:

``` text
IDisposable
     +
SafeHandle
```

rather than manually implementing a finalizer.

------------------------------------------------------------------------

## 7. WeakReference

### Q29. What problem does WeakReference solve?

**Answer:**

A `WeakReference` allows an object to be referenced without keeping that
object alive for GC purposes.

This can be useful for caches and other scenarios where an object can be
recreated if necessary.

Conceptually:

``` text
Strong reference
      ↓
Object remains alive

Weak reference
      ↓
Object may be collected
```

------------------------------------------------------------------------

### Q30. If an object is referenced only through WeakReference, will GC keep it alive?

**Answer:**

No.

A weak reference does not constitute the same kind of strong
reachability that keeps an object alive.

If no strong references remain, the object may be collected.

------------------------------------------------------------------------

### Q31. Why might a cache use weak references?

**Answer:**

A cache may want to reuse objects when they are available but should not
prevent GC from reclaiming them when memory pressure exists.

This creates a trade-off:

``` text
Strong cache
→ Better retention
→ Higher memory usage

Weak cache
→ Lower retention pressure
→ Objects may disappear
→ Objects may need to be recreated
```

------------------------------------------------------------------------

## 8. Closures and Object Lifetime

### Q32. What does a lambda expression capture?

Consider:

``` csharp
List<Action> actions = new();

for (int i = 0; i < 1000; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
```

**Answer:**

The lambda captures the variable according to C# closure semantics.

The compiler generates internal closure/state objects to represent
captured state.

Therefore, captured variables can have a lifetime that extends beyond
the original source-level scope.

------------------------------------------------------------------------

### Q33. Can closures cause objects to remain alive longer than expected?

**Answer:**

Yes.

If a closure captures an object and the delegate remains reachable, the
captured object can remain reachable as well.

Conceptually:

``` text
Long-lived delegate
       ↓
Closure object
       ↓
Captured object
```

This can create unexpected memory retention.

------------------------------------------------------------------------

## 9. Async/Await and Memory

### Q34. Does async/await create objects on the heap?

**Answer:**

It can.

The compiler transforms async methods into state-machine machinery.
Depending on the method, execution path, and runtime optimizations,
state can be represented using heap allocations.

The key interview point is:

> **`async/await` is not simply "another thread"; it transforms method
> execution into resumable state.**

------------------------------------------------------------------------

### Q35. What happens to local variables across an await?

Consider:

``` csharp
async Task Process()
{
    var largeObject = new byte[100_000_000];

    await SomeOperationAsync();

    Console.WriteLine(largeObject.Length);
}
```

**Answer:**

If `largeObject` is required after the `await`, its state must remain
available when the asynchronous operation resumes.

The async state-machine representation can therefore keep the relevant
reference alive across the suspension point.

This means a large object can remain alive while the async operation is
waiting if the state machine still needs it.

------------------------------------------------------------------------

### Q36. Can async code contribute to memory retention?

**Answer:**

Yes.

Potential causes include:

-   Large objects captured across `await`
-   Long-running tasks
-   Uncompleted tasks
-   Large state machines
-   Queued continuations
-   Incorrect cancellation handling

A good diagnostic question is:

> **What references are keeping this object alive while the asynchronous
> operation is waiting?**

------------------------------------------------------------------------

## 10. Boxing and Unboxing

### Q37. What happens here?

``` csharp
int x = 10;
object obj = x;
```

**Answer:**

The value type `int` is boxed.

Conceptually:

``` text
int value
   ↓
boxing
   ↓
object on managed heap
```

Boxing creates a managed object containing the value.

------------------------------------------------------------------------

### Q38. What happens during this operation?

``` csharp
int y = (int)obj;
```

**Answer:**

The boxed value is unboxed and converted back to the value type.

Unboxing requires the object to contain the appropriate boxed value
type.

------------------------------------------------------------------------

### Q39. Can excessive boxing cause performance problems?

**Answer:**

Yes.

Repeated boxing can produce many temporary heap allocations, increasing:

-   Allocation rate
-   GC pressure
-   CPU usage

Generic APIs often help avoid unnecessary boxing:

``` csharp
List<int>
```

is generally preferable to designs that repeatedly treat integers as
`object`.

------------------------------------------------------------------------

## 11. Structs and Classes

### Q40. Does using a struct guarantee stack allocation?

**Answer:**

No.

This is a common interview trap.

A value type does not automatically mean "stack allocated."

A struct can exist:

-   As a local
-   Inline inside another object
-   Inside an array
-   In registers
-   As a boxed object on the managed heap

------------------------------------------------------------------------

### Q41. Can a struct be allocated on the managed heap?

**Answer:**

Yes.

For example:

``` csharp
MyStruct value = new MyStruct();
object obj = value;
```

Assigning the struct to `object` causes **boxing**, which creates an
object on the managed heap containing the boxed value.

------------------------------------------------------------------------

### Q42. Why is "value type = stack, reference type = heap" an incorrect explanation?

**Answer:**

Because storage location depends on context.

The simplified rule:

``` text
value type → stack
reference type → heap
```

is useful for beginners but technically incorrect.

A better explanation is:

> **Value/reference type describes type semantics and how values are
> represented and passed; it does not by itself determine whether
> storage is on the stack or heap.**

------------------------------------------------------------------------

## 12. Span`<T>`{=html} and ref struct

### Q43. Why is Span`<T>`{=html} a ref struct?

**Answer:**

`Span<T>` can refer to memory that may have stack-based lifetime.

A `ref struct` has restrictions designed to prevent unsafe lifetime
escapes.

For example, it cannot generally be:

-   Stored as a field of a normal heap object
-   Boxed
-   Used across an `await`
-   Used in contexts that could allow it to outlive the memory it
    references

These restrictions protect memory safety.

------------------------------------------------------------------------

### Q44. Why can't Span`<T>`{=html} normally be stored as a field of a class?

**Answer:**

A normal class is heap allocated and can live much longer than
stack-based memory referenced by a `Span<T>`.

Allowing unrestricted storage could create a dangling reference to stack
memory.

Therefore `Span<T>` is a `ref struct`, and the compiler enforces
restrictions that prevent unsafe lifetime extension.

------------------------------------------------------------------------

### Q45. Why can't a Span`<T>`{=html} normally cross an await boundary?

Example:

``` csharp
async Task Process()
{
    Span<int> values = stackalloc int[100];

    await Task.Delay(100);
}
```

**Answer:**

The async method can suspend and resume later.

A stack-based span could refer to memory whose lifetime does not extend
across that suspension.

Therefore C# restricts `ref struct` types such as `Span<T>` from
crossing async/iterator boundaries.

------------------------------------------------------------------------

## 13. stackalloc

### Q46. Where is memory allocated by stackalloc?

``` csharp
Span<int> buffer = stackalloc int[1000];
```

**Answer:**

The storage is allocated in stack memory associated with the current
execution.

The allocation is tied to the lifetime of the relevant stack
frame/execution.

When that execution ends, the stack storage is no longer valid.

------------------------------------------------------------------------

### Q47. What happens when a method using stackalloc returns?

**Answer:**

The stack allocation is automatically invalidated as the stack frame is
unwound.

You should not allow a reference to that stack memory to escape its
valid lifetime.

This is one reason the type system places restrictions around `Span<T>`
and `ref struct`.

------------------------------------------------------------------------

## 14. Pinning

### Q48. What does fixed do?

Example:

``` csharp
fixed (byte* p = buffer)
{
    // use p
}
```

**Answer:**

`fixed` pins the referenced managed object so that the GC cannot move
that object during the pinned region.

This is important when unmanaged code requires a stable memory address.

------------------------------------------------------------------------

### Q49. Why can pinning hurt GC performance?

**Answer:**

A compacting GC normally moves objects to reduce fragmentation.

Pinned objects cannot be moved.

Too much or long-lived pinning can therefore:

-   Restrict compaction
-   Increase fragmentation
-   Make allocation more difficult
-   Reduce GC efficiency

Therefore pinning should be used carefully.

------------------------------------------------------------------------

### Q50. Why does the GC move objects?

**Answer:**

A compacting GC can move live objects so that free memory becomes
contiguous.

For example:

``` text
Before:

[Live][Free][Live][Free][Live]

After compaction:

[Live][Live][Live][Free][Free]
```

This reduces fragmentation and makes future allocations easier.

------------------------------------------------------------------------

# 15. Production Memory Investigation

### Q51. An application has 500 MB managed heap, frequent Gen 2 collections, and increasing memory usage. How would you investigate?

**Answer:**

Do not immediately conclude that there is a memory leak.

First determine whether the problem is:

-   Excessive allocation
-   Object retention
-   LOH growth
-   LOH fragmentation
-   Excessive Gen 2 promotion
-   Unmanaged/native memory growth
-   Legitimate application growth

A useful investigation path is:

``` text
Memory growth
     ↓
Is managed memory growing?
     ↓
Allocation rate
     ↓
GC frequency
     ↓
Gen 0 / Gen 1 / Gen 2 behavior
     ↓
Retained objects
     ↓
GC roots
     ↓
LOH
     ↓
Native/unmanaged memory
```

Useful tools include:

-   Visual Studio Diagnostic Tools
-   `dotnet-counters`
-   `dotnet-gcdump`
-   `dotnet-dump`
-   PerfView
-   JetBrains dotMemory

------------------------------------------------------------------------

# 16. Advanced Scenario Questions

## Q52. Consider this code. Is there a memory leak?

``` csharp
public class CustomerService
{
    private readonly List<byte[]> _cache = new();

    public void Process()
    {
        _cache.Add(new byte[10_000_000]);
    }
}
```

**Answer:**

Potentially yes, as a logical memory-retention problem.

If `Process()` is called repeatedly, the list keeps strong references to
every array.

Therefore:

``` text
CustomerService
      ↓
   _cache
      ↓
List<byte[]>
      ↓
byte[] byte[] byte[] ...
```

The GC sees these objects as reachable.

Therefore GC cannot reclaim them.

The problem is not that GC failed. The problem is that the application
continues to maintain references to objects that are no longer useful.

------------------------------------------------------------------------

## Q53. If an object is no longer needed by the business, why doesn't GC collect it?

**Answer:**

Because GC does not understand business semantics.

It does not know:

> "This customer is no longer required."

It only knows:

> "There is still a reachable reference to this object."

Therefore, memory management problems often require examining the
**reference graph**, not merely the allocation code.

------------------------------------------------------------------------

## Q54. If you call GC.Collect(), will that solve a memory leak?

**Answer:**

Usually no.

`GC.Collect()` can force a collection, but if objects are still
reachable, they will not be collected.

For example:

``` csharp
_cache.Add(largeObject);
GC.Collect();
```

The object remains reachable through `_cache`.

Therefore:

> **GC.Collect() cannot solve a reachability problem.**

It may also hurt performance if used unnecessarily.

------------------------------------------------------------------------

## Q55. What is the difference between allocation rate and memory retention?

**Answer:**

**Allocation rate** is how quickly the application creates objects.

**Memory retention** is how much memory remains occupied by objects that
are still reachable.

An application can have:

``` text
High allocation + low retention
```

and still experience high GC activity.

Another application can have:

``` text
Low allocation + high retention
```

and steadily consume memory.

This distinction is extremely useful during production diagnosis.

------------------------------------------------------------------------

# 17. Architect-Level Scenario

## Q56. An application memory usage grows continuously. How would you determine whether it is a leak, excessive allocation, LOH fragmentation, or legitimate growth?

**Answer:**

I would first separate **managed heap growth** from **process/native
memory growth**.

Then I would investigate:

1.  Allocation rate
2.  GC frequency
3.  Gen 0/1/2 behavior
4.  Gen 2 promotion
5.  Object retention
6.  Large Object Heap usage
7.  GC roots keeping objects alive
8.  Native/unmanaged allocations
9.  Application caches
10. Event subscriptions and long-lived references

The key question is:

> **Which objects are consuming memory, and what is keeping them
> reachable?**

For retained managed objects, a memory profiler can show the object
graph and GC roots.

For runtime-level analysis, tools such as `dotnet-counters`,
`dotnet-gcdump`, `dotnet-dump`, PerfView, Visual Studio diagnostics, and
dotMemory can be useful.

------------------------------------------------------------------------

# 18. High-Value Interview Traps

## Q57. Is every object created with `new` placed on the heap?

**Answer:**

For ordinary reference-type objects, `new` creates an object on the
managed heap.

However, the statement should not be generalized to every C# construct
because allocation and storage are affected by type semantics and
runtime/JIT optimizations.

For example, value types can be stored in different locations depending
on context.

------------------------------------------------------------------------

## Q58. Is every local variable stored on the stack?

**Answer:**

No.

The compiler/JIT determines the actual representation.

A local may be stored in:

-   A stack location
-   A register
-   A field of a compiler-generated state/closure object

Therefore, "local variable = stack" is also an oversimplification.

------------------------------------------------------------------------

## Q59. Does null mean the object is destroyed?

**Answer:**

No.

`null` means a particular reference no longer refers to an object.

The object itself may still be referenced elsewhere.

Only when there are no relevant strong paths from GC roots can it become
eligible for collection.

------------------------------------------------------------------------

## Q60. Does Dispose mean the object is dead?

**Answer:**

No.

An object can still be referenced after `Dispose()`.

`Dispose()` indicates that the object's owned resources should be
released and that the object should generally no longer be used.

Its managed memory is still subject to normal GC rules.

------------------------------------------------------------------------

# 19. The 10 Questions You Should Definitely Prepare

For a Senior/Lead/Architect-level C# interview, prioritize these:

### 1. GC Roots

**Question:** How does the GC determine whether an object is alive?

**Core answer:** By determining reachability from GC roots.

### 2. Object Lifetime

**Question:** Does going out of scope immediately destroy an object?

**Core answer:** No. It may become unreachable, but GC is
nondeterministic.

### 3. Managed Memory Leak

**Question:** How can C# have a memory leak if it has GC?

**Core answer:** Objects can remain reachable through unintended
references.

### 4. Generational GC

**Question:** Why does .NET use Gen 0, Gen 1, and Gen 2?

**Core answer:** Most objects die young, so collecting younger
generations more frequently improves efficiency.

### 5. LOH

**Question:** Why can a process have high memory usage even after
objects are collected?

**Core answer:** The runtime can retain committed heap memory for reuse,
and LOH/fragmentation can also contribute.

### 6. IDisposable

**Question:** Why do we need IDisposable when GC exists?

**Core answer:** GC manages managed memory, while IDisposable provides
deterministic cleanup of resources.

### 7. Finalizer

**Question:** Why are finalizers expensive?

**Core answer:** Finalizable objects require special processing and
generally survive longer before reclamation.

### 8. Closure

**Question:** Can a closure extend an object's lifetime?

**Core answer:** Yes. A reachable delegate can keep its closure and
captured objects alive.

### 9. Async

**Question:** Can an object remain alive across await?

**Core answer:** Yes, if the async state machine needs the reference
after the suspension point.

### 10. Production Diagnosis

**Question:** How would you diagnose a memory leak in production?

**Core answer:** Measure allocation and heap growth, identify retained
objects, inspect GC roots, investigate LOH/Gen 2 behavior, and
distinguish managed from native memory.

------------------------------------------------------------------------

# 20. Final Interview Principle

The most important mental model is:

``` text
              GC ROOTS
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Object A             Object B
        │
        ↓
   Object C
```

The GC asks:

> **Can I reach this object from a GC root?**

If yes:

``` text
Reachable → Live → Not collectible
```

If no:

``` text
Unreachable → Eligible for GC → Eventually reclaimable
```

Therefore, the most important sentence to remember for a C#
memory-management interview is:

> **The .NET GC does not collect objects because they are no longer
> logically needed; it collects objects that are no longer reachable
> from GC roots.**

That principle explains:

-   Garbage collection
-   Generations
-   Memory retention
-   Managed memory leaks
-   Event-related leaks
-   Static references
-   Caches
-   Closures
-   Async state
-   Weak references
-   Object lifetime
-   Much of production memory diagnosis
