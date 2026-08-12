# Large Object Heap (LOH)

## 1. Introduction

So far, we have established:

```text
Memory Allocation
      ↓
Object Lifetime
      ↓
GC Roots & Reachability
      ↓
Garbage Collection
      ↓
Generational GC
      ↓
Gen 0 → Gen 1 → Gen 2
```

We now need to understand one more important part of the managed heap:

> **What happens when an object is very large?**

.NET has a special area called the:

> **Large Object Heap (LOH)**

The LOH is used for allocating large managed objects.

---

# 2. What Is the Large Object Heap?

The **Large Object Heap (LOH)** is an area of the managed heap used for large objects.

For example:

```csharp
byte[] buffer = new byte[100_000];
```

If the object is large enough to meet the LOH allocation threshold, it is allocated on the LOH rather than the normal small-object allocation area.

Conceptually:

```text
Managed Heap
+--------------------------------------+
| Small Object Heap | Large Object Heap|
|                   |                  |
| Gen 0             |                  |
| Gen 1             |                  |
| Gen 2             |      LOH         |
+--------------------------------------+
```

The exact physical organization of the managed heap is more sophisticated than this diagram.

This diagram is only a conceptual model.

---

# 3. Why Does .NET Have an LOH?

Large objects require different memory-management considerations.

For example:

```csharp
byte[] image = new byte[10_000_000];
```

Moving a very large object during every small garbage collection could be expensive.

Therefore, .NET treats large objects differently from ordinary small allocations.

The important idea is:

```text
Small objects
     ↓
Normal managed heap allocation

Large objects
     ↓
Large Object Heap
```

---

# 4. What Kind of Objects Go to the LOH?

A common example is a large array.

For example:

```csharp
byte[] buffer = new byte[1_000_000];
```

Other examples can include large arrays of:

```text
int[]
double[]
object[]
```

and large objects whose size exceeds the LOH allocation threshold.

The important point is:

> **LOH allocation is based on the size of the object, not simply on the type of the object.**

For example, both of these are arrays:

```csharp
byte[] smallBuffer = new byte[100];

byte[] largeBuffer = new byte[1_000_000];
```

The small array may be allocated in the normal managed heap, while the sufficiently large array may be allocated on the LOH.

---

# 5. The Approximate LOH Threshold

A commonly used rule for .NET is:

> **Objects of approximately 85,000 bytes or larger are allocated on the LOH.**

This is an important interview number.

Remember:

```text
~85 KB
   ↓
LOH threshold
```

However, this should be understood as an approximate runtime threshold rather than a universal rule that every object of exactly 85,000 bytes will always behave identically.

The actual allocation decision depends on the runtime and object layout.

For interview purposes:

> **Approximately 85,000 bytes is the traditional LOH threshold.**

---

# 6. Example — Small Array

Consider:

```csharp
byte[] data = new byte[10_000];
```

The array is relatively small.

Conceptually:

```text
Managed Heap

Small Object Area
      |
      +----> byte[10_000]
```

It does not meet the traditional LOH threshold.

---

# 7. Example — Large Array

Now consider:

```csharp
byte[] data = new byte[1_000_000];
```

This is a large object.

Conceptually:

```text
Managed Heap

Large Object Heap
      |
      +----> byte[1_000_000]
```

The large array is allocated on the LOH.

---

# 8. Is the Array Reference Stored on the LOH?

This is an important distinction.

Consider:

```csharp
byte[] data = new byte[1_000_000];
```

There are two different things to think about:

```text
data
```

and:

```text
byte[1_000_000]
```

The array object itself is the large object.

Conceptually:

```text
Reference
   |
   v
Large byte[] object
   |
   v
LOH
```

The important rule is:

> **The LOH contains the large object itself, not a separate copy of the reference variable.**

As we discussed earlier, the location of a reference variable depends on where that variable is stored.

The object it refers to is what is allocated on the LOH.

---

# 9. LOH and Generations

The LOH is closely associated with **Gen 2** garbage collection.

A useful conceptual model is:

```text
Small Object Heap
    |
    +---- Gen 0
    |
    +---- Gen 1
    |
    +---- Gen 2


Large Object Heap
    |
    +---- associated with Gen 2
```

The LOH is not simply another generation:

```text
LOH ≠ Gen 3
```

There is no:

```text
Gen 3
```

The LOH is a separate heap area that is collected as part of Gen 2 collections.

---

# 10. Why Is LOH Associated with Gen 2?

Large objects are generally treated as long-lived candidates from the GC's perspective.

The LOH is therefore collected during a **Gen 2 garbage collection**.

Conceptually:

```text
Gen 0 collection
      ↓
Gen 0

Gen 1 collection
      ↓
Gen 0 + Gen 1

Gen 2 collection
      ↓
Gen 0 + Gen 1 + Gen 2
      +
     LOH
```

This is a simplified conceptual model.

The actual GC behavior depends on the runtime and collection mode.

The key point to remember is:

> **The LOH is collected as part of Gen 2 garbage collection.**

---

# 11. What Happens When a Large Object Becomes Unreachable?

Consider:

```csharp
byte[] buffer = new byte[1_000_000];
```

Conceptually:

```text
GC Root
   |
   v
buffer
   |
   v
Large byte[] object
   |
   v
LOH
```

Now:

```csharp
buffer = null;
```

Assume there are no other references.

The object becomes unreachable:

```text
GC Roots

(no path)

Large byte[] object
        X
       LOH
```

The object is now eligible for garbage collection.

When an appropriate collection occurs, its memory can be reclaimed.

---

# 12. LOH and Garbage Collection

The overall process is:

```text
Large object created
        ↓
Allocated on LOH
        ↓
Object remains reachable
        ↓
Object becomes unreachable
        ↓
Object becomes eligible for GC
        ↓
Gen 2 collection
        ↓
LOH processed
        ↓
Memory can be reclaimed
```

This connects the LOH to everything we have already learned.

---

# 13. Why Can LOH Fragmentation Be a Problem?

Consider:

```text
LOH

[A][B][C][D][E]
```

Suppose `B` and `D` become unreachable:

```text
[A][free][C][free][E]
```

Now the LOH contains free regions between live objects.

This is called:

> **Fragmentation**

Conceptually:

```text
Live object
    ↓
Free space
    ↓
Live object
    ↓
Free space
    ↓
Live object
```

The free memory exists, but it may not form one large contiguous region.

---

# 14. Why Is Fragmentation Important for Large Objects?

Suppose the application needs another large object:

```csharp
byte[] buffer = new byte[2_000_000];
```

The runtime may need a sufficiently large contiguous region for the allocation.

If free space is fragmented:

```text
[Large Object][free][Large Object][free][Large Object]
```

the available free memory may not be arranged ideally for the new allocation.

Therefore:

> **LOH fragmentation can affect allocation efficiency and memory usage.**

---

# 15. Does the LOH Always Compact?

Historically, the LOH was generally not compacted during ordinary garbage collections because moving large objects can be expensive.

Modern .NET also provides mechanisms for LOH compaction when explicitly requested.

The important distinction is:

```text
Normal GC
    ↓
LOH is generally not compacted automatically
in the same way as the small object heap
```

But:

```text
LOH compaction
    ↓
Can be requested when appropriate
```

For example, .NET provides:

```csharp
GCSettings.LargeObjectHeapCompactionMode
```

which can be used to request LOH compaction.

The application should not casually force LOH compaction.

It is a specialized operation that can have a performance cost.

---

# 16. LOH Allocation and Performance

Large allocations can be expensive.

Consider:

```csharp
for (int i = 0; i < 1000; i++)
{
    byte[] buffer = new byte[1_000_000];
}
```

This creates many large objects.

Conceptually:

```text
Large allocation
       ↓
LOH
       ↓
Large allocation
       ↓
LOH
       ↓
Large allocation
       ↓
LOH
       ↓
...
```

If these objects become unreachable frequently, the application can generate significant GC and memory-management activity.

Therefore:

> **Repeated large allocations should be considered carefully in performance-sensitive applications.**

---

# 17. LOH Does Not Mean "Never Collected"

A common misconception is:

> "Once an object goes to the LOH, it stays there forever."

False.

An LOH object can become unreachable.

For example:

```text
LOH
 |
 +----> Large byte[]
```

If the last reference disappears:

```text
GC Roots

(no path)

Large byte[]
```

the object becomes eligible for collection.

During an appropriate Gen 2 collection, its memory can be reclaimed.

---

# 18. LOH Does Not Mean "Gen 3"

Another common misconception is:

```text
Gen 0
Gen 1
Gen 2
Gen 3 = LOH
```

This is incorrect.

There is no Gen 3 representing the LOH.

Instead:

```text
Generations
    |
    +---- Gen 0
    +---- Gen 1
    +---- Gen 2


Large Object Heap
    |
    +---- separate heap area
    +---- associated with Gen 2 collection
```

---

# 19. LOH and Object References

Consider:

```csharp
class ImageData
{
    public byte[] Pixels;
}
```

And:

```csharp
ImageData image = new ImageData();
image.Pixels = new byte[1_000_000];
```

Conceptually:

```text
GC Root
   |
   v
ImageData
   |
   | Pixels
   v
Large byte[]
   |
   v
LOH
```

Notice that there are two objects:

```text
ImageData
byte[]
```

The `ImageData` object itself may be a normal managed object.

The large `byte[]` can be allocated on the LOH.

This is another important example of why we must distinguish:

> **The containing object from the object referenced by one of its fields.**

---

# 20. Example — Large Field vs Large Object

Consider:

```csharp
class Image
{
    public byte[] Data;
}
```

The class definition itself does not mean:

```text
Image → LOH
```

Instead, when we create:

```csharp
Image image = new Image();
image.Data = new byte[1_000_000];
```

we have:

```text
Image object
    |
    +---- Data
            |
            v
       Large byte[]
            |
            v
           LOH
```

The `Image` object and the large array are two separate objects.

The array is the large object that qualifies for LOH allocation.

---

# 21. LOH and the Managed Heap

A useful conceptual picture is:

```text
                 MANAGED MEMORY

        +----------------------------+
        | Small Object Heap          |
        |                            |
        | Gen 0                      |
        | Gen 1                      |
        | Gen 2                      |
        +----------------------------+

        +----------------------------+
        | Large Object Heap          |
        |                            |
        | Large objects              |
        +----------------------------+
```

Again, this is a simplified learning model.

The actual .NET runtime's heap organization is more sophisticated.

---

# 22. Complete Example

Consider:

```csharp
byte[] image = new byte[1_000_000];
```

### Step 1 — Allocation

A large array is created.

```text
Large byte[]
      ↓
LOH
```

### Step 2 — Reference

The variable refers to the object:

```text
GC Root
   |
   v
image
   |
   v
Large byte[]
```

### Step 3 — Object remains reachable

As long as a path from a GC root exists:

```text
GC Root
   ↓
Large byte[]
```

the object remains reachable.

### Step 4 — Reference disappears

```csharp
image = null;
```

Assume no other references exist.

Now:

```text
GC Roots

(no path)

Large byte[]
```

### Step 5 — Object becomes eligible

```text
Large byte[]
      ↓
Eligible for GC
```

### Step 6 — Gen 2 collection

The LOH is processed as part of the appropriate Gen 2 collection.

### Step 7 — Memory reclaimed

The memory occupied by the unreachable large object can be reclaimed.

---

# 23. Common Misconceptions

### Misconception 1

> "Every large object automatically goes to LOH regardless of its exact size."

Not necessarily.

The traditional threshold is approximately **85,000 bytes**, and the actual runtime allocation behavior depends on object layout and runtime implementation.

---

### Misconception 2

> "LOH is Gen 3."

False.

LOH is not a generation.

It is a separate heap area associated with Gen 2 collection.

---

### Misconception 3

> "LOH objects are never garbage collected."

False.

Unreachable LOH objects can be collected.

---

### Misconception 4

> "Every array is allocated on the LOH."

False.

Only sufficiently large objects qualify.

```text
Small array → normal managed heap

Large array → potentially LOH
```

---

### Misconception 5

> "If a class contains a large array, the entire class object moves to LOH."

False.

For example:

```csharp
class Image
{
    public byte[] Data;
}
```

The `Image` object and `Data` array are separate objects.

The large `Data` array may be on the LOH while the `Image` object itself is elsewhere in the managed heap.

---

# 24. Final Mental Model

Remember this:

```text
                  OBJECT CREATED
                        |
                        v
                Is the object large?
                    /       \
                  No         Yes
                  |           |
                  v           v
             Normal Heap      LOH
                  |           |
                  |           |
                  +-----+-----+
                        |
                        v
                  Object reachable
                        |
                        v
               Object becomes
                unreachable
                        |
                        v
                  Eligible for GC
                        |
                        v
                 Appropriate GC
                        |
                        v
              Memory can be reclaimed
```

And remember the relationship with generations:

```text
Small Object Heap
    ↓
Gen 0 → Gen 1 → Gen 2

Large Object Heap
    ↓
Collected with Gen 2
```

---

# 25. One Sentence to Remember

> **The Large Object Heap (LOH) is a special area of the .NET managed heap used for sufficiently large objects, traditionally around 85,000 bytes or more, and it is collected as part of Gen 2 garbage collection.**

The next natural topic is:

```text
LOH
 ↓
Heap Fragmentation
 ↓
Compaction
 ↓
Why memory may remain allocated
even after objects are collected
```

This will help explain an important real-world question:

> **"If the GC collected my objects, why does the process still appear to use a lot of memory?"**
