# .NET CLR Garbage Collection — Memory Lifecycle

## 1. Overview

The .NET CLR uses a **Garbage Collector (GC)** to automatically manage memory for managed objects.

The most important distinction is:

> An object becoming unreachable does not immediately mean its memory is removed.

The lifecycle is:

```text
Object Allocation
      ↓
Object becomes reachable
      ↓
Object becomes unreachable
      ↓
Object becomes eligible for GC
      ↓
GC collection starts
      ↓
Find live objects
      ↓
Identify garbage
      ↓
Move / compact surviving objects (when compaction is performed)
      ↓
Contiguous free space is created
      ↓
CLR reuses the free space
      ↓
New objects can be allocated
      ↓
Some unused heap memory may eventually be released to the OS
```

---

# 2. Object Allocation

When an application creates a managed object:

```csharp
var customer = new Customer();
```

The CLR allocates memory for the object on the **managed heap**.

New objects are normally allocated in **Generation 0 (Gen 0)**.

Conceptually:

```text
Managed Heap

┌──────────┬──────────┬──────────┐
│ Customer │ Object B │ Object C │
└──────────┴──────────┴──────────┘
     ↑
   Gen 0
```

---

# 3. Object Is Reachable

An object is considered alive when it can be reached from a **GC root**.

Examples of GC roots include:

- Local variables that are still live
- Static references
- Active method arguments
- References held by runtime structures
- Other GC roots maintained by the runtime

Conceptually:

```text
GC Root
   │
   ▼
customer
   │
   ▼
Customer Object
```

As long as the object is reachable, GC must preserve it.

---

# 4. Object Becomes Unreachable

Suppose the reference is removed:

```csharp
customer = null;
```

And assume there are no other references to the object.

Now:

```text
GC Root
   │
   ▼
 null


Customer Object

(no path from a GC root)
```

The object is now **unreachable**.

It is therefore:

> **Eligible for garbage collection**

However, it is **not immediately deleted**.

Its memory is still occupied in the managed heap until a GC collection processes it.

---

# 5. GC Is Triggered

Eventually, the CLR performs a garbage collection.

A collection can happen for several reasons, including allocation pressure.

Conceptually:

```text
Allocation pressure
       ↓
GC starts
```

The exact behavior depends on the generation being collected and the GC configuration.

---

# 6. GC Identifies Live Objects and Garbage

During collection, the GC determines which objects are reachable and therefore must survive.

For example:

```text
Before GC:

┌────┬───────┬────┬───────┬────┐
│ A  │ DEAD  │ B  │ DEAD  │ C  │
└────┴───────┴────┴───────┴────┘
```

The GC determines:

```text
A     → LIVE
B     → LIVE
C     → LIVE

DEAD  → GARBAGE
```

A useful mental model is:

```text
LIVE OBJECTS
    ↓
must be preserved

GARBAGE
    ↓
storage can eventually be reclaimed
```

---

# 7. Compaction Moves Live Objects

For a compacting collection, the GC moves surviving objects together.

The garbage itself is not moved as an object.

Before compaction:

```text
┌────┬───────┬────┬───────┬────┐
│ A  │ DEAD  │ B  │ DEAD  │ C  │
└────┴───────┴────┴───────┴────┘
```

The GC moves the live objects:

```text
┌────┬────┬────┬────────────────┐
│ A  │ B  │ C  │      FREE      │
└────┴────┴────┴────────────────┘
```

The important point is:

> **The live objects are moved. The garbage's old storage becomes part of the free region.**

---

# 8. Why Does GC Compact the Heap?

Without compaction, the heap could contain fragmented free space:

```text
┌────┬──────┬────┬──────┬────┬──────┐
│ A  │ FREE │ B  │ FREE │ C  │ FREE │
└────┴──────┴────┴──────┴────┴──────┘
```

There is free memory, but it is split into multiple holes.

After compaction:

```text
┌────┬────┬────┬────────────────────┐
│ A  │ B  │ C  │        FREE        │
└────┴────┴────┴────────────────────┘
```

Now the free space is contiguous.

This makes future allocations easier and reduces fragmentation.

---

# 9. References Must Be Updated When Objects Move

If an object moves during compaction, references to that object must continue to point to its new location.

Conceptually:

Before:

```text
Object X
   │
   └──────────► Object C
```

After `Object C` is moved:

```text
Object X
   │
   └──────────► Object C (new location)
```

The CLR's GC handles the necessary reference updates.

Application code does not manually update these references.

---

# 10. Reclamation of Garbage

After the surviving objects have been handled, the storage previously occupied by garbage is available as free space.

For example:

```text
Before:

[A][DEAD][B][DEAD][C]

After:

[A][B][C][FREE][FREE]
```

There is no C# operation such as:

```csharp
delete object;
```

for normal managed objects.

Instead, the GC determines that the object is unreachable and reclaims its storage as part of the collection process.

---

# 11. Reclaimed Space Can Be Reused

This is an important distinction.

Suppose the managed heap now looks like:

```text
┌────┬────┬────┬────────────────┐
│ A  │ B  │ C  │      FREE      │
└────┴────┴────┴────────────────┘
```

The CLR can use that free space for a future allocation:

```csharp
var order = new Order();
```

Conceptually:

```text
┌────┬────┬────┬───────┬────────┐
│ A  │ B  │ C  │ Order │  FREE  │
└────┴────┴────┴───────┴────────┘
```

Therefore:

> **GC-reclaimed memory can be reused by the CLR for future managed allocations.**

It does not have to request additional memory from the OS immediately.

---

# 12. Object Promotion

Objects that survive collections can be promoted between generations.

The simplified generational model is:

```text
Gen 0
  │
  │ survives collection
  ▼
Gen 1
  │
  │ survives collection
  ▼
Gen 2
```

Conceptually:

```text
New object
    ↓
  Gen 0
    ↓ survives
  Gen 1
    ↓ survives
  Gen 2
```

The purpose is to avoid repeatedly processing long-lived objects during collections focused on younger generations.

The exact physical layout and movement depend on the GC implementation and collection type.

---

# 13. Important: Promotion and Compaction Are Different Concepts

Do not treat these as the same operation.

### Promotion

Promotion means:

> An object survived a collection and is considered part of an older generation.

```text
Gen 0 → Gen 1 → Gen 2
```

### Compaction

Compaction means:

> Surviving objects are moved together to reduce fragmentation and create contiguous free space.

```text
Before:

[A][FREE][B][FREE][C]

After:

[A][B][C][FREE][FREE]
```

An object may be promoted and may also be moved as part of heap organization, but these are conceptually different GC actions.

---

# 14. Does GC Immediately Return Reclaimed Memory to the OS?

No.

This is one of the most important distinctions.

After GC, suppose:

```text
Managed Heap

┌────┬────┬────┬────────────────┐
│ A  │ B  │ C  │      FREE      │
└────┴────┴────┴────────────────┘
```

The CLR can keep the free region as part of its managed heap.

The CLR can then reuse it:

```text
Future allocation
       ↓
uses existing free heap space
```

Therefore:

```text
GC reclaims object storage
          ≠
immediately returning memory to OS
```

---

# 15. What Does "Return Memory to the OS" Mean?

The CLR obtains memory from the operating system for the process.

Conceptually:

```text
Operating System
       │
       ▼
      CLR
       │
       ▼
Managed Heap
       │
       ▼
Managed Objects
```

If the CLR has heap memory that it no longer needs, it may release some of that memory back to the OS.

Conceptually:

```text
Before:

OS
 │
 ▼
CLR
 │
 └── Managed Heap: 500 MB
       ├── Live objects: 100 MB
       └── Unused: 400 MB
```

The runtime may release some unused heap memory:

```text
OS
 │
 ▼
CLR
 │
 └── Managed Heap: smaller amount
       └── Live objects
```

The released memory can then be used by the operating system for other purposes.

However, this is a separate decision from simply reclaiming an individual object's storage.

---

# 16. The Most Important Distinction

Keep these concepts separate:

```text
OBJECT BECOMES UNREACHABLE
          ↓
OBJECT IS ELIGIBLE FOR GC
          ↓
GC IDENTIFIES IT AS GARBAGE
          ↓
STORAGE IS RECLAIMED
          ↓
SPACE CAN BE REUSED BY CLR
          ↓
[OPTIONALLY]
SOME UNUSED HEAP MEMORY MAY BE
RELEASED TO THE OS
```

These statements are not equivalent:

```text
Unreachable
    ≠
Collected
    ≠
Returned to OS
```

---

# 17. Complete Simplified Lifecycle

This is the lifecycle to remember:

```text
┌──────────────────────┐
│ 1. OBJECT ALLOCATED  │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 2. OBJECT IS LIVE    │
│    / REACHABLE       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 3. OBJECT BECOMES    │
│    UNREACHABLE       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 4. ELIGIBLE FOR GC   │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 5. GC COLLECTION     │
│    STARTS            │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 6. FIND LIVE OBJECTS │
│    AND GARBAGE       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 7. HANDLE SURVIVORS  │
│    MOVE / COMPACT    │
│    IF REQUIRED       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 8. GARBAGE STORAGE   │
│    IS RECLAIMED      │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 9. FREE SPACE IS     │
│    AVAILABLE         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 10. CLR REUSES IT    │
│     FOR ALLOCATION   │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ 11. SOME UNUSED HEAP │
│     MAY BE RELEASED  │
│     TO THE OS        │
└──────────────────────┘
```

---

# 18. Interview Answer

If asked:

> **"Explain the .NET GC memory lifecycle."**

A concise but accurate answer is:

> A managed object is allocated on the managed heap, normally starting in Gen 0. As long as the object is reachable from a GC root, it remains alive. When it becomes unreachable, it becomes eligible for garbage collection, but it is not immediately removed. When GC runs, it identifies reachable objects and garbage. For a compacting collection, surviving objects may be moved together to eliminate fragmentation, while the storage occupied by garbage becomes free space. Surviving objects may also be promoted to older generations. The reclaimed space can then be reused by the CLR for future allocations. Returning heap memory to the operating system is a separate decision and does not necessarily happen immediately after GC.

---

# 19. Final Mental Model

Remember this:

```text
ALLOCATE
   ↓
LIVE / REACHABLE
   ↓
UNREACHABLE
   ↓
ELIGIBLE FOR GC
   ↓
GC IDENTIFIES LIVE + GARBAGE
   ↓
MOVE / COMPACT SURVIVORS
   ↓
GARBAGE STORAGE BECOMES FREE
   ↓
CLR REUSES FREE SPACE
   ↓
OPTIONALLY RELEASE UNUSED HEAP MEMORY TO OS
```

## The Three Most Important Rules

1. **Unreachable does not mean immediately deleted.**

2. **GC-reclaimed space normally becomes reusable managed-heap space.**

3. **Reclaimed managed-heap space is not the same thing as memory being returned to the OS.**
