# C# Memory Management — Complete Mental Model and Interview Revision

## 1. The Complete Flow

The topics can now be connected into one mental model:

```text
Variable
   ↓
Value or Reference
   ↓
Object Allocation
   ↓
Object Lifetime
   ↓
Reachability
   ↓
GC Roots
   ↓
Garbage Collection
   ↓
Generations
   ↓
LOH
   ↓
Fragmentation / Compaction
   ↓
Pinned Objects
   ↓
IDisposable
   ↓
Unmanaged Resources
   ↓
Finalizers / SafeHandle
   ↓
Memory Leaks
   ↓
Diagnostics
```

---

# 2. Variables, Values, and References

Remember:

```text
Value-type variable
    ↓
Contains a value

Reference-type variable
    ↓
Contains a reference
    ↓
Reference points to an object
```

Do not reduce this to:

```text
Value type = stack
Reference type = heap
```

That is too simplistic.

A value-type field is stored inline inside its containing object.

A reference-type field stores a reference inline inside its containing object.

---

# 3. Object Allocation

Example:

```csharp
Person p = new Person();
```

Conceptually:

```text
p
 |
 v
Person object
```

The variable and object are distinct concepts.

---

# 4. Reachability

The GC asks whether an object is reachable from GC roots.

```text
GC Root
   |
   v
Object A
   |
   v
Object B
```

Both are reachable.

If the root loses the reference:

```text
GC Root

(no path)

Object A
   |
   v
Object B
```

the graph can become unreachable.

---

# 5. Garbage Collection

The GC reclaims memory occupied by unreachable managed objects.

Important distinction:

```text
Unreachable
    ↓
Eligible for collection
```

does not mean:

```text
Unreachable
    ↓
Immediately destroyed
```

---

# 6. Generational GC

New objects generally start in Gen 0.

Objects that survive collections can be promoted.

```text
Gen 0
  ↓
Gen 1
  ↓
Gen 2
```

Many short-lived objects die in Gen 0.

Long-lived objects may reach Gen 2.

---

# 7. LOH

Sufficiently large objects are allocated on the Large Object Heap.

Traditional interview threshold:

```text
~85,000 bytes
```

The LOH is associated with Gen 2 collection.

It is not Gen 3.

---

# 8. Fragmentation

After garbage is reclaimed:

```text
[Live][Free][Live][Free][Live]
```

free memory can become fragmented.

Compaction can move surviving objects:

```text
[Live][Live][Live][Free][Free]
```

---

# 9. Pinned Objects

Pinned objects cannot be freely moved during the pinning period.

```text
Pinned object
      ↓
Cannot move
      ↓
Can interfere with compaction
```

Pinning is useful for native interop but should not be overused.

Modern .NET also has a Pinned Object Heap for certain pinned allocations.

---

# 10. `IDisposable`

The GC manages managed memory.

`Dispose()` provides deterministic cleanup for resources owned by an object.

```text
Resource acquired
      ↓
Use
      ↓
Dispose()
      ↓
Resource released
```

Calling `Dispose()` does not directly destroy the managed object.

---

# 11. `using`

`using` expresses a deterministic lifetime boundary:

```csharp
using (var stream = File.OpenRead("data.txt"))
{
    Process(stream);
}
```

Conceptually:

```text
Create
  ↓
Use
  ↓
Dispose
```

---

# 12. Unmanaged Resources

Examples include:

- OS handles
- Native memory
- Native library resources
- Certain external resources

A managed object can wrap such a resource:

```text
Managed wrapper
      |
      v
Unmanaged resource
```

The wrapper may implement `IDisposable`.

---

# 13. Finalizers

Finalizers provide non-deterministic fallback cleanup for types that require finalization semantics.

```text
Dispose()
   ↓
Deterministic cleanup

Finalizer
   ↓
Fallback cleanup
```

Do not treat finalizers as a replacement for `Dispose()`.

---

# 14. SafeHandle

For unmanaged handles, `SafeHandle` provides a safer ownership abstraction.

```text
Managed owner
      |
      v
SafeHandle
      |
      v
Native handle
```

It reduces the need for custom low-level finalization logic.

---

# 15. Memory Leaks

A .NET memory leak often occurs when an object remains reachable even though the application no longer needs it.

Typical causes include:

```text
Static collections
Event subscriptions
Timers
Callbacks
Unbounded caches
Closures
Growing collections
```

The key diagnostic question is:

> Why is this object still reachable?

---

# 16. Diagnostics

When memory is high:

```text
Do not guess
     ↓
Measure
     ↓
Identify object types
     ↓
Check allocation rate
     ↓
Find retention paths
     ↓
Find GC roots
     ↓
Understand ownership/lifetime
     ↓
Fix
     ↓
Measure again
```

---

# 17. Interview Questions to Be Able to Answer

### Question 1

**Are value types always stored on the stack?**

No.

Storage depends on context. A value-type field is stored inline inside its containing object.

---

### Question 2

**Are reference variables always stored on the stack?**

No.

The location of a reference variable depends on where that variable is declared and stored.

---

### Question 3

**Where is a reference-type field stored?**

The reference itself is stored inline inside its containing storage/object.

The referenced object is a separate object.

---

### Question 4

**What makes an object eligible for GC?**

It becomes unreachable from GC roots.

---

### Question 5

**Does unreachable mean immediately collected?**

No.

It means eligible for collection.

---

### Question 6

**What are Gen 0, Gen 1, and Gen 2?**

They are generations used by the GC to optimize collection based on object survival.

---

### Question 7

**Why does an object move from Gen 0 to Gen 1?**

Because it survived a collection and is treated as a longer-lived object.

---

### Question 8

**Does every object reach Gen 2?**

No.

Many objects die in Gen 0.

---

### Question 9

**What is the LOH?**

The Large Object Heap is a special heap area used for sufficiently large objects.

---

### Question 10

**Is LOH Gen 3?**

No.

The LOH is not a generation. It is associated with Gen 2 collection.

---

### Question 11

**What is fragmentation?**

Free memory exists in separated regions rather than being conveniently consolidated.

---

### Question 12

**What is compaction?**

Moving surviving objects so that free memory becomes more contiguous.

---

### Question 13

**Why can the GC not always move an object?**

Pinned objects cannot be freely moved during their pinning period.

---

### Question 14

**Does `Dispose()` destroy an object?**

No.

It releases resources according to the object's disposal contract.

---

### Question 15

**Does GC automatically call `Dispose()` for every disposable object?**

Do not rely on that.

Use deterministic disposal through the owner or appropriate framework lifetime management.

---

### Question 16

**Why do we need finalizers?**

They provide fallback finalization for types that require it, especially around unmanaged-resource ownership.

---

### Question 17

**Are finalizers deterministic?**

No.

They are GC-driven.

---

### Question 18

**What is a memory leak in .NET?**

Typically, unnecessary objects remain reachable and therefore cannot be collected.

---

# 18. Final One-Page Mental Model

```text
                 C# MEMORY MANAGEMENT

                         |
                         v
                 MEMORY ALLOCATION
                         |
              +----------+----------+
              |                     |
          VALUE TYPE          REFERENCE TYPE
              |                     |
        value stored         reference stored
                                  |
                                  v
                              OBJECT
                                  |
                                  v
                           OBJECT LIFETIME
                                  |
                                  v
                            REACHABILITY
                                  |
                                  v
                              GC ROOTS
                                  |
                       +----------+----------+
                       |                     |
                   Reachable             Unreachable
                       |                     |
                       v                     v
                  Keep alive            Eligible for GC
                                             |
                                             v
                                      GARBAGE COLLECTION
                                             |
                                             v
                                      GENERATIONAL GC
                                             |
                              +--------------+--------------+
                              |              |              |
                            Gen 0          Gen 1          Gen 2
                                                             |
                                                             v
                                                            LOH

                    Fragmentation / Compaction
                              |
                              v
                       Pinned Objects

                 RESOURCE MANAGEMENT
                              |
                  +-----------+-----------+
                  |                       |
              Managed memory        External resources
                  |                       |
                  v                       v
                  GC                 IDisposable
                                          |
                                          v
                                        using
                                          |
                                          v
                                      Dispose()
                                          |
                                          v
                                     SafeHandle
                                          |
                                          v
                                      Finalization

                 MEMORY PROBLEMS
                              |
                 +------------+------------+
                 |                         |
             Retention                 Fragmentation
                 |                         |
                 v                         v
            Memory leak             Allocation pressure
                 |
                 v
              Diagnostics
```

---

# 19. Final Principle

The most important memory-management principle is:

> **Understand ownership, reachability, and lifetime.**

If you understand:

```text
Who owns this object?
        ↓
Who references this object?
        ↓
When should it stop being reachable?
        ↓
Does it own an external resource?
        ↓
When should that resource be released?
```

you can reason about most C# memory-management problems without memorizing implementation details.
