# Garbage Collection (GC)

## 1. Introduction

In the previous chapters, we established the following flow:

```text
Object Creation
      ↓
Object Lifetime
      ↓
GC Roots
      ↓
Reachability
      ↓
Object becomes unreachable
      ↓
Eligible for GC
```

The next question is:

> **What actually happens when the .NET Garbage Collector runs?**

This chapter introduces the basic working of Garbage Collection before moving into more advanced topics such as generations and the Large Object Heap.

---

# 2. What Is Garbage Collection?

**Garbage Collection (GC)** is the .NET runtime mechanism responsible for automatically managing the memory used by managed objects.

When an object is no longer reachable from the GC roots, its memory can eventually be reclaimed by the garbage collector.

Conceptually:

```text
Object created
      ↓
Object is reachable
      ↓
Object becomes unreachable
      ↓
GC identifies it
      ↓
Memory is reclaimed
```

The important point is:

> **The programmer normally does not explicitly free the memory occupied by managed objects.**

The .NET runtime manages this memory automatically.

---

# 3. Why Does .NET Need Garbage Collection?

Consider:

```csharp
Person p = new Person();
```

A `Person` object requires memory.

Later:

```csharp
p = null;
```

If there are no other references to the object, the object becomes unreachable.

Without automatic memory management, the programmer would have to explicitly determine when that memory should be released.

The .NET GC handles this automatically.

Conceptually:

```text
Application
    |
    v
Create objects
    |
    v
Use objects
    |
    v
Objects become unreachable
    |
    v
Garbage Collector
    |
    v
Memory becomes available again
```

---

# 4. What Does the GC Actually Do?

At a high level, the garbage collector performs three important activities:

```text
1. Identify reachable objects
2. Reclaim memory from unreachable objects
3. Compact memory when appropriate
```

A simplified model is:

```text
GC starts
   ↓
Find reachable objects
   ↓
Identify unreachable objects
   ↓
Reclaim their memory
   ↓
Compact remaining objects when appropriate
```

The actual .NET GC is considerably more sophisticated than this simplified model, but this model is useful for understanding the fundamentals.

---

# 5. Marking — Identifying Reachable Objects

The first important concept is determining which objects are still reachable.

Suppose we have:

```text
GC Root
   |
   v
Object A
   |
   v
Object B
```

and:

```text
Object C
```

If there is no path from a GC root to `Object C`:

```text
GC Root
   |
   v
Object A
   |
   v
Object B


Object C
```

then:

```text
Object A → reachable
Object B → reachable
Object C → unreachable
```

Conceptually, the GC identifies the reachable objects and distinguishes them from objects that are no longer reachable.

This is commonly described as the **marking** phase.

---

# 6. Reclaiming Memory

Once unreachable objects have been identified, the memory occupied by those objects can be reclaimed.

For example:

```text
Before GC:

Object A
Object B
Object C
Object D
```

Suppose:

```text
Object A → reachable
Object B → reachable
Object C → unreachable
Object D → unreachable
```

The memory occupied by `Object C` and `Object D` can be reclaimed.

Conceptually:

```text
Before:

[A][B][C][D]

After reclaiming:

[A][B][free][free]
```

The reclaimed memory can then be used for future allocations.

---

# 7. Why Is Compaction Needed?

Simply reclaiming memory can create gaps.

For example:

```text
[A][free][B][free][C]
```

These gaps are called **fragmentation**.

Even though there is free memory, it may be spread across multiple locations.

A compacting collection can move live objects closer together:

```text
Before:

[A][free][B][free][C]


After compaction:

[A][B][C][free][free]
```

This creates a larger contiguous free region.

---

# 8. Can the GC Move Objects?

Yes.

A garbage collector can move managed objects during compaction.

For example:

```text
Before:

[A][free][B][free][C]
```

After compaction:

```text
[A][B][C][free][free]
```

`B` and `C` have moved.

This is possible because managed references are tracked by the runtime.

The important idea is:

> **Managed objects can be relocated by the GC during compaction.**

This is one reason managed references should not be treated like ordinary raw memory addresses.

---

# 9. What Happens to References When an Object Moves?

Suppose:

```text
Person
   ^
   |
p
```

The GC moves the `Person` object during compaction.

Conceptually:

```text
Before:

p
 |
 v
Person object
```

After moving the object:

```text
p
 |
 v
new location of Person object
```

The runtime updates the relevant managed references so that they continue to refer to the correct object.

Therefore application code normally does not need to know that the object moved.

This is one of the important benefits of managed memory.

---

# 10. GC Does Not Simply Delete Every Object

The GC does **not** delete objects merely because it is running.

It distinguishes between:

```text
Reachable objects
        and
Unreachable objects
```

For example:

```text
GC Root
   |
   v
Object A
   |
   v
Object B

Object C
```

During collection:

```text
Object A → live
Object B → live
Object C → collectible
```

The GC preserves reachable objects and reclaims memory associated with objects that are no longer reachable.

---

# 11. When Does Garbage Collection Run?

A common misconception is:

> "The GC runs whenever an object becomes unreachable."

That is not correct.

An object can become unreachable at one moment:

```text
Object becomes unreachable
        ↓
Eligible for collection
```

The GC may run later.

Conceptually:

```text
Object becomes unreachable
        ↓
Eligible for GC
        ↓
Application continues running
        ↓
GC decides collection is appropriate
        ↓
Collection occurs
```

The exact decision is made by the runtime based on factors such as memory pressure and GC heuristics.

Therefore:

> **Becoming unreachable and being collected are two different events.**

---

# 12. Eligible for GC vs Actually Collected

This distinction is extremely important.

Suppose:

```csharp
Person p = new Person();

p = null;
```

Assume no other references exist.

The object becomes unreachable.

That means:

```text
Person object
      ↓
Eligible for GC
```

It does **not** mean:

```text
Person object
      ↓
Immediately destroyed
```

The actual sequence is:

```text
Reference removed
      ↓
Object becomes unreachable
      ↓
Object becomes eligible for GC
      ↓
GC eventually runs
      ↓
Memory is reclaimed
```

---

# 13. Does the Programmer Normally Control GC?

Normally, no.

The .NET runtime manages garbage collection automatically.

Application code generally creates and uses objects:

```csharp
Person person = new Person();
```

and the runtime determines when memory should be reclaimed.

The programmer should generally focus on:

- avoiding unnecessary object retention
- releasing unmanaged resources appropriately
- avoiding accidental long-lived references
- designing efficient object lifetimes

The GC itself is controlled by the runtime.

---

# 14. What About `GC.Collect()`?

.NET provides:

```csharp
GC.Collect();
```

which can explicitly request a garbage collection.

However, application code should **not normally call `GC.Collect()` as a routine memory-management technique**.

Why?

Because the runtime has its own GC heuristics and knows about the application's allocation and memory behavior.

Forcing collections unnecessarily can introduce performance overhead.

Therefore:

> **Do not use `GC.Collect()` simply because an object has become unreachable.**

The normal model is:

```text
Object becomes unreachable
        ↓
Runtime determines when GC should run
```

---

# 15. GC and Application Performance

Garbage collection is automatic, but it is not free.

A collection requires the runtime to perform work such as:

- identifying reachable objects
- reclaiming memory
- potentially compacting memory
- updating references when objects move

Therefore excessive allocation or poor object-lifetime management can contribute to GC overhead.

For example, code that continuously creates large numbers of short-lived objects may create significant allocation and collection activity.

Conceptually:

```text
Many allocations
      ↓
More garbage
      ↓
More GC work
      ↓
Potential performance impact
```

This does not mean:

> "Never create objects."

It means:

> **Understand allocation patterns and object lifetimes when performance matters.**

---

# 16. Managed Memory vs Unmanaged Memory

The .NET GC manages **managed objects**.

For example:

```csharp
Person person = new Person();
```

The memory for the managed `Person` object is handled by the runtime.

However, applications can also work with **unmanaged resources**.

Examples include:

- operating system handles
- file handles
- native resources
- unmanaged memory

The GC does not replace the need to manage such resources correctly.

This distinction leads to an important principle:

```text
Managed memory
      ↓
GC manages object memory


Unmanaged resources
      ↓
Application must use the appropriate
resource-management mechanism
```

This is why concepts such as `IDisposable`, `Dispose()`, and `using` are important.

They should be studied separately from ordinary managed-object garbage collection.

---

# 17. A Complete Example

Consider:

```csharp
void Test()
{
    Person person = new Person();
}
```

### Step 1 — Object creation

```text
person
   |
   v
Person object
```

### Step 2 — Object is reachable

The active reference provides a path to the object.

```text
GC Root
   |
   v
Person object
```

### Step 3 — Method finishes

Assume no other references exist.

```text
GC Roots

(no path)

Person object
```

The object is now unreachable.

### Step 4 — Object becomes eligible

```text
Person object
      ↓
Eligible for GC
```

### Step 5 — GC eventually runs

The GC identifies the object as unreachable.

```text
GC
 ↓
Object identified as unreachable
```

### Step 6 — Memory is reclaimed

```text
Person object
      ↓
Memory reclaimed
```

The entire lifecycle can therefore be represented as:

```text
new Person()
     ↓
Object created
     ↓
Object reachable
     ↓
Reference disappears
     ↓
Object becomes unreachable
     ↓
Eligible for GC
     ↓
GC runs
     ↓
Memory reclaimed
```

---

# 18. Another Example — Multiple Objects

Consider:

```csharp
Person p = new Person();
p.Address = new Address();
```

Conceptually:

```text
GC Root
   |
   v
Person
   |
   v
Address
```

Both objects are reachable.

Now:

```csharp
p = null;
```

Assume there are no other references.

```text
GC Roots

(no path)

Person
   |
   v
Address
```

Both objects are unreachable.

When the GC eventually processes them, the memory occupied by both objects can be reclaimed.

---

# 19. The Basic GC Model

At a simplified conceptual level:

```text
                 GARBAGE COLLECTION

                        |
                        v
                 Find GC Roots
                        |
                        v
               Follow references
                        |
                        v
               Identify reachable
                    objects
                        |
                        v
             Identify unreachable
                    objects
                        |
                        v
              Reclaim their memory
                        |
                        v
             Compact when appropriate
```

This is a conceptual model.

The real .NET garbage collector contains many optimizations and sophisticated algorithms, which we will study in later chapters.

---

# 20. Important Distinctions

Keep these distinctions clear.

### Object becomes unreachable

```text
No path from a GC root
```

### Object becomes eligible for GC

```text
Unreachable object can be reclaimed
```

### GC runs

```text
Runtime performs a collection
```

### Memory is reclaimed

```text
Memory occupied by collectible objects
becomes available for reuse
```

These are related but **not identical events**.

---

# 21. What We Have Learned So Far

The memory-management story is now:

```text
1. Object is created
        ↓
2. Reference points to object
        ↓
3. Object is reachable
        ↓
4. References may disappear
        ↓
5. Object becomes unreachable
        ↓
6. Object becomes eligible for GC
        ↓
7. GC eventually runs
        ↓
8. Unreachable object's memory is reclaimed
        ↓
9. Live objects may be compacted
```

This gives us the foundation needed to understand the next major concept:

> **Why does .NET divide objects into generations?**

---

# 22. Final Mental Model

Remember this simple picture:

```text
                GC ROOT
                   |
                   v
               Object A
                   |
                   v
               Object B


               Object C
```

`Object A` and `Object B` are reachable.

`Object C` is not reachable if there is no path from any GC root.

Therefore:

```text
Object A → remains alive
Object B → remains alive
Object C → eligible for GC
```

Later, when the GC runs:

```text
Reachable objects
      ↓
preserved

Unreachable objects
      ↓
memory reclaimed

Live objects
      ↓
may be compacted
```

## One Sentence to Remember

> **Garbage collection identifies objects that are no longer reachable from GC roots, reclaims the memory associated with those objects, and may compact the remaining managed objects to reduce fragmentation.**

This is the foundation for the next topic:

```text
Garbage Collection
        ↓
Generational Garbage Collection
        ↓
Gen 0 → Gen 1 → Gen 2
        ↓
Large Object Heap (LOH)
```
