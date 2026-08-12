# C# Memory Management — Memory Leaks in .NET

## 1. Introduction

A common misconception is:

> "C# has garbage collection, so memory leaks cannot happen."

That is false.

Garbage collection prevents many manual-memory-management errors, but applications can still retain objects unnecessarily.

A .NET memory leak often means:

> **Objects remain reachable even though the application no longer logically needs them.**

---

# 2. How Can a GC-Based Application Leak Memory?

The GC can only collect unreachable objects.

If an unnecessary object is still reachable:

```text
GC Root
   |
   v
Unwanted object
```

the GC must treat it as live.

Therefore:

```text
Unwanted
   +
Reachable
   =
Memory retained
```

---

# 3. Static References

A static collection can keep objects alive for the lifetime of the process.

Example:

```csharp
static List<Person> cache = new();
```

If objects are continuously added and never removed:

```text
Static root
    |
    v
List
    |
    +----> Person
    +----> Person
    +----> Person
    +----> ...
```

those objects remain reachable.

---

# 4. Event Handler Leaks

Events can create unintended object lifetimes.

Conceptually:

```text
Long-lived publisher
       |
       v
Event subscription
       |
       v
Short-lived subscriber
```

If the subscriber is expected to die but remains subscribed to a long-lived publisher, the publisher may keep the subscriber reachable.

Therefore:

> Event subscription can become an ownership/lifetime issue.

---

# 5. Timers and Callbacks

Timers, callbacks, background operations, and registrations can also keep objects alive.

Conceptually:

```text
Long-lived timer
      |
      v
Callback
      |
      v
Object
```

If the callback captures or references an object longer than intended, the object may remain reachable.

---

# 6. Caches

Caches are a common source of accidental retention.

A cache is intentionally designed to keep objects alive.

The problem occurs when the cache grows without bounds.

```text
Cache
 |
 +----> Object
 +----> Object
 +----> Object
 +----> ...
```

A cache should have an intentional eviction/lifetime policy when appropriate.

---

# 7. Closures

A lambda or closure can capture variables.

For example:

```csharp
Action action = () =>
{
    Console.WriteLine(largeObject);
};
```

If `action` remains alive, the captured object can remain reachable.

Conceptually:

```text
Long-lived delegate
       |
       v
Closure
       |
       v
Large object
```

Therefore closures can affect object lifetime.

---

# 8. Collections That Keep Growing

A simple pattern:

```csharp
private readonly List<object> _items = new();
```

If objects are continuously added:

```text
List
 |
 +----> Object
 +----> Object
 +----> Object
 +----> ...
```

the list can become a long-term retention point.

---

# 9. Memory Leak vs Fragmentation

These are different.

### Leak

```text
GC Root
   |
   v
Unwanted object
```

The object is reachable.

### Fragmentation

```text
Live
Free
Live
Free
Live
```

Memory exists but is divided into regions.

Therefore:

```text
Memory leak
    ≠
Fragmentation
```

---

# 10. Memory Leak vs High Memory Usage

High process memory does not automatically mean a leak.

Possible reasons include:

- The application legitimately has many live objects
- The runtime has retained heap segments
- The LOH contains allocated regions
- Native memory is being used
- Fragmentation exists
- A real retention problem exists

Diagnosis requires evidence.

---

# 11. Common Leak Patterns

Watch for:

```text
Static collections
Event subscriptions
Timers
Callbacks
Long-lived caches
Closures
Unbounded collections
Incorrect resource ownership
```

---

# 12. How to Think About a Suspected Leak

Ask:

1. What object is growing?
2. Who references it?
3. What is the GC root?
4. Why does that root still reference it?
5. Is that retention intentional?
6. Where should the lifetime have ended?

The key question is:

> **Why is this object still reachable?**

---

# 13. Common Misconceptions

### Misconception 1

> "GC means memory leaks are impossible."

False.

GC only collects unreachable objects.

---

### Misconception 2

> "A memory leak always means unmanaged memory."

False.

Managed objects can also be retained unnecessarily.

---

### Misconception 3

> "High process memory proves a leak."

False.

High memory usage requires investigation.

---

# 14. Final Mental Model

```text
Object created
      ↓
Application no longer needs it
      ↓
But something still references it
      ↓
Object remains reachable
      ↓
GC cannot collect it
      ↓
Memory retained
```

## One Sentence to Remember

> **A .NET memory leak commonly occurs when objects remain reachable through unintended references even though the application no longer needs them.**
