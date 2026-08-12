# C# Memory Management — Heap Fragmentation and Compaction

## 1. Introduction

Garbage collection does not simply mean:

```text
Garbage disappears
↓
All memory becomes one continuous block
```

Live objects may exist between objects that have become unreachable.

This can create **fragmentation**.

The GC can sometimes improve the layout by moving live objects together. This process is called **compaction**.

---

# 2. What Is Heap Fragmentation?

Consider:

```text
[A][B][C][D][E]
```

Suppose `B` and `D` become unreachable:

```text
[A][free][C][free][E]
```

There is free memory, but it is separated into multiple regions.

This is fragmentation.

## Key Idea

> Fragmentation means that available free memory is divided into separate regions rather than being one convenient contiguous region.

---

# 3. Why Does Fragmentation Matter?

Suppose we need a large contiguous allocation.

We might have:

```text
[Live][Free][Live][Free][Live]
```

There may be enough total free memory, but it may not be arranged as one sufficiently large region.

This can make allocations less efficient.

---

# 4. What Is Compaction?

Compaction means moving live objects so that free space becomes more contiguous.

Before:

```text
[A][free][C][free][E]
```

After compaction:

```text
[A][C][E][free][free]
```

The live objects have been moved together.

The free memory is now consolidated.

---

# 5. Why Can the GC Move Objects?

Managed references are controlled by the runtime.

When an object moves, the GC can update references that point to the moved object.

Conceptually:

```text
Before:

Reference
   |
   v
   A


After:

Reference
   |
   v
   A
```

The object's physical location may change while the program's logical reference remains valid.

This is one reason managed memory can be compacted safely.

---

# 6. A Simple Example

Suppose:

```text
Managed Heap

[A][B][C][D][E]
```

`B` and `D` become unreachable:

```text
[A][free][C][free][E]
```

The GC identifies live objects:

```text
A
C
E
```

It can compact them:

```text
[A][C][E][free][free]
```

Now the free space is consolidated.

---

# 7. Compaction Is Not the Same as Collection

These are different concepts.

### Collection

Identifies unreachable objects and reclaims their memory.

### Compaction

Moves surviving objects to reduce fragmentation.

Therefore:

```text
Collection
    ↓
Reclaim garbage

Compaction
    ↓
Rearrange surviving objects
```

They can occur together, but they are conceptually different operations.

---

# 8. Why Does Compaction Have a Cost?

Moving objects requires work.

Suppose we have:

```text
Object A
Object B
Object C
...
Object Z
```

If many live objects must be moved, the runtime has to:

1. Move the objects
2. Update references
3. Maintain heap metadata

Therefore compaction is not free.

The GC balances:

```text
Memory efficiency
        vs
CPU / pause cost
```

---

# 9. Small Object Heap and Compaction

The Small Object Heap (SOH) is generally designed to benefit from compaction.

A simplified model:

```text
SOH

[A][free][C][free][E]
          ↓
      Compaction
          ↓
[A][C][E][free][free]
```

This helps reduce fragmentation.

The actual .NET GC uses sophisticated algorithms and different collection modes, so this diagram is only a learning model.

---

# 10. LOH and Compaction

The LOH is different.

Large objects can be expensive to move.

Historically, LOH compaction was not performed as part of ordinary collections in the same way as the small object heap.

Modern .NET provides a mechanism to request LOH compaction when appropriate.

For example:

```csharp
GCSettings.LargeObjectHeapCompactionMode
```

can be used to request LOH compaction.

The important idea is:

```text
SOH
↓
Compaction is a normal part of GC behavior

LOH
↓
Compaction is more expensive and is handled differently
```

---

# 11. Why Can Objects Be Moved Safely?

Consider:

```csharp
Person p = new Person();
Person another = p;
```

Conceptually:

```text
p -------+
         |
         v
      Person
         ^
         |
another--+
```

If the GC moves the `Person` object:

```text
p -------+
         |
         v
      Person
         ^
         |
another--+
```

the runtime updates the references to the object's new location.

The application does not have to manually update those references.

---

# 12. Pinned Objects

There is an important exception.

Sometimes managed code or native interop needs an object to remain at a stable memory address.

Such an object can be **pinned**.

Conceptually:

```text
Object
   |
   v
PINNED
   |
   X
Cannot be moved during that period
```

Pinned objects can interfere with compaction.

This is why pinning is an important memory-management topic.

We will study it separately.

---

# 13. Fragmentation and Pinned Objects

Suppose:

```text
[A][B][C][D][E]
```

`B` is pinned and `C` becomes garbage:

```text
[A][B pinned][free][D][E]
```

The GC cannot freely move `B` during the pinning period.

Therefore the free space may remain separated.

This can contribute to fragmentation.

Conceptually:

```text
Live
Pinned
Free
Live
Live
```

instead of:

```text
Live
Live
Live
Free
Free
```

---

# 14. Does Garbage Collection Always Compact?

No.

A collection and compaction are not identical.

The runtime decides how to perform a collection based on:

- Generation
- GC mode
- Heap state
- Object layout
- Runtime heuristics
- Pinning
- Other performance considerations

Therefore:

> Do not assume that every GC automatically compacts every heap region.

---

# 15. Why Memory Can Still Look High After GC

This is a common real-world question.

Suppose objects become unreachable:

```text
Object A → garbage
Object B → garbage
Object C → garbage
```

The GC can reclaim their space.

But the process may still appear to use significant memory.

Why?

Because:

```text
Managed memory reclaimed
        ≠
Operating system immediately receives every byte back
```

The runtime may retain heap segments for future allocations.

Therefore:

```text
Objects collected
        ↓
Memory becomes available to the managed heap
        ↓
Process working set may still remain high
```

This is not automatically a memory leak.

---

# 16. Memory Reuse

Suppose the GC has reclaimed space:

```text
[Live][Free][Free][Live]
```

The runtime can reuse that free memory for future managed allocations.

Therefore memory can be available to the application even if the operating system's process-level memory number does not immediately decrease.

---

# 17. Fragmentation vs Memory Leak

These are different problems.

### Fragmentation

Memory is available, but free space is divided or poorly arranged.

### Memory Leak

Objects remain reachable even though the application no longer logically needs them.

For example:

```text
Root
 |
 v
Collection
 |
 +----> Object that should have been released
```

Because the object is still reachable, the GC cannot collect it.

Therefore:

```text
Fragmentation
    ≠
Memory Leak
```

---

# 18. A Practical Example

Consider an application processing images.

```csharp
byte[] image = LoadLargeImage();
Process(image);
image = null;
```

The large array may be allocated on the LOH.

After `image = null`, if no other references exist, it becomes unreachable.

Later a Gen 2 collection can reclaim it.

But the LOH may now look like:

```text
[Large Live Object][Free][Large Live Object][Free]
```

If many allocations and releases occur, fragmentation can develop.

---

# 19. Why Reusing Large Buffers Can Help

Instead of repeatedly allocating large arrays:

```csharp
for (...)
{
    byte[] buffer = new byte[1_000_000];
    Process(buffer);
}
```

an application may sometimes benefit from reusing buffers.

Conceptually:

```text
Create buffer once
      ↓
Reuse buffer
      ↓
Reuse buffer
      ↓
Reuse buffer
```

This can reduce repeated large allocations and associated GC pressure.

The correct approach depends on the application and concurrency requirements.

---

# 20. Common Misconceptions

### Misconception 1

> "GC always compacts the entire heap."

False.

Compaction depends on the GC and the heap region being collected.

---

### Misconception 2

> "Collection and compaction are the same thing."

False.

Collection reclaims garbage.

Compaction rearranges surviving objects to reduce fragmentation.

---

### Misconception 3

> "If memory is freed, the process memory must immediately decrease."

False.

The runtime may retain memory for future allocations.

---

### Misconception 4

> "Fragmentation means there is a memory leak."

False.

Fragmentation and leaks are different problems.

---

### Misconception 5

> "The GC can always move every object."

False.

Pinned objects are an important exception.

---

# 21. Final Mental Model

```text
Objects allocated
       ↓
Objects become unreachable
       ↓
GC identifies garbage
       ↓
Garbage memory can be reclaimed
       ↓
Free regions may exist
       ↓
Fragmentation
       ↓
Surviving objects may be moved
       ↓
Compaction
       ↓
Free space becomes more contiguous
```

With pinning:

```text
Pinned object
      ↓
Cannot be freely moved
      ↓
Can interfere with compaction
      ↓
Can contribute to fragmentation
```

---

# 22. One Sentence to Remember

> **Fragmentation occurs when free memory is divided into separate regions, while compaction moves surviving objects together so that free memory becomes more contiguous.**
