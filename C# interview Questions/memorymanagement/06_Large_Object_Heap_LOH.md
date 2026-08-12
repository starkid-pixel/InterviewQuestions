# C# Memory Management — Large Object Heap (LOH)

## 1. Introduction

The .NET runtime has a special area called the:

> Large Object Heap (LOH)

The LOH is used for sufficiently large managed objects.

## 2. What Is the LOH?

The LOH is an area of the managed heap used for large objects.

For example:

```csharp
byte[] buffer = new byte[1_000_000];
```

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

This is a conceptual model; the runtime's physical organization is more sophisticated.

## 3. Why Does .NET Have an LOH?

Large objects require different memory-management considerations.

Moving a very large object during every small collection could be expensive.

Therefore .NET treats large objects differently from ordinary small allocations.

## 4. What Kind of Objects Go to the LOH?

A common example is a large array:

```csharp
byte[] buffer = new byte[1_000_000];
```

Other examples include sufficiently large arrays such as:

```text
int[]
double[]
object[]
```

LOH allocation is based on object size, not simply on its type.

## 5. Approximate Threshold

A traditional interview rule is:

> Objects of approximately 85,000 bytes or larger are allocated on the LOH.

Treat this as an approximate runtime threshold rather than an exact universal boundary for every possible object layout.

## 6. Small vs Large Array

```csharp
byte[] smallBuffer = new byte[100];

byte[] largeBuffer = new byte[1_000_000];
```

The small array may be allocated on the normal managed heap.

The sufficiently large array may be allocated on the LOH.

## 7. Is the Reference Stored on the LOH?

Consider:

```csharp
byte[] data = new byte[1_000_000];
```

There are two separate concepts:

```text
data
```

and:

```text
byte[1_000_000]
```

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

The large object itself is allocated on the LOH. The reference variable is not a separate copy stored there.

## 8. LOH and Generations

The LOH is closely associated with Gen 2 collection.

```text
Small Object Heap
    |
    +---- Gen 0
    +---- Gen 1
    +---- Gen 2

Large Object Heap
    |
    +---- associated with Gen 2 collection
```

The LOH is not Gen 3.

## 9. What Happens When a Large Object Becomes Unreachable?

```csharp
byte[] buffer = new byte[1_000_000];
buffer = null;
```

Assuming no other references exist:

```text
GC Roots

(no path)

Large byte[] object
        X
       LOH
```

The object is eligible for garbage collection.

## 10. LOH and Garbage Collection

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

## 11. LOH Fragmentation

Suppose:

```text
LOH

[A][B][C][D][E]
```

Objects `B` and `D` become unreachable:

```text
[A][free][C][free][E]
```

Free regions now exist between live objects.

This is fragmentation.

## 12. Why Fragmentation Matters

Suppose the application needs another large object:

```csharp
byte[] buffer = new byte[2_000_000];
```

Free memory may not be arranged ideally for the allocation.

Therefore:

> LOH fragmentation can affect allocation efficiency and memory usage.

## 13. LOH Compaction

Historically, the LOH was generally not compacted during ordinary collections because moving large objects can be expensive.

Modern .NET provides mechanisms for requesting LOH compaction when appropriate.

For example:

```csharp
GCSettings.LargeObjectHeapCompactionMode
```

can be used to request LOH compaction.

Compaction should not be forced casually because it can have a performance cost.

## 14. LOH Allocation and Performance

Repeated large allocations can create memory-management pressure.

```csharp
for (int i = 0; i < 1000; i++)
{
    byte[] buffer = new byte[1_000_000];
}
```

If these objects become unreachable frequently, the application can generate significant allocation and GC activity.

## 15. LOH Does Not Mean "Never Collected"

An LOH object can become unreachable and can eventually be collected.

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

## 16. LOH Does Not Mean "Gen 3"

Incorrect:

```text
Gen 0
Gen 1
Gen 2
Gen 3 = LOH
```

Correct:

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

## 17. LOH and Object References

Consider:

```csharp
class ImageData
{
    public byte[] Pixels;
}

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

There are two objects:

```text
ImageData
byte[]
```

The `ImageData` object itself may be a normal managed object.

The large `byte[]` can be allocated on the LOH.

## 18. Large Field vs Large Object

The class definition does not mean the entire object goes to LOH.

```csharp
class Image
{
    public byte[] Data;
}
```

When:

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

The `Image` object and the large array are separate objects.

## 19. Final Mental Model

```text
                  OBJECT CREATED
                        |
                        v
                Is the object large?
                    /                         No         Yes
                  |           |
                  v           v
             Normal Heap      LOH
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

Relationship with generations:

```text
Small Object Heap
    ↓
Gen 0 → Gen 1 → Gen 2

Large Object Heap
    ↓
Collected with Gen 2
```

## One Sentence to Remember

> The Large Object Heap (LOH) is a special area of the .NET managed heap used for sufficiently large objects, traditionally around 85,000 bytes or more, and it is collected as part of Gen 2 garbage collection.
