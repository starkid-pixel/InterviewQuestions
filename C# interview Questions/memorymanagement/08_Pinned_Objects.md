# C# Memory Management — Pinned Objects

## 1. Introduction

The .NET GC can normally move managed objects during compaction.

This allows it to reduce fragmentation.

But sometimes an object must remain at a stable memory address.

This is called **pinning**.

---

# 2. What Is a Pinned Object?

A pinned object is a managed object that the GC is temporarily prevented from moving.

Conceptually:

```text
Object
   |
   v
Pinned
   |
   X
Cannot move during pinning
```

Pinning is usually needed when managed memory must interact with unmanaged or native code that expects a stable address.

---

# 3. Why Would an Object Need a Stable Address?

Suppose native code receives a pointer to memory.

```text
Managed object
      |
      v
Memory address
      |
      v
Native code
```

If the GC moved the object while native code was using that pointer:

```text
Old address
     X
```

the native pointer could become invalid.

Pinning prevents the GC from moving the object during the required period.

---

# 4. Example Using `fixed`

For unsafe code, an object can be pinned using `fixed`.

For example:

```csharp
byte[] buffer = new byte[100];

unsafe
{
    fixed (byte* p = buffer)
    {
        // Native/unsafe operation using p
    }
}
```

During the `fixed` block, the array is pinned.

Conceptually:

```text
GC
 |
 +----> byte[] buffer
           |
           v
         PINNED
           |
           X
        cannot move
```

When the `fixed` block ends, the pinning ends.

---

# 5. Pinning Is Usually Temporary

A good rule is:

> Pin only for as long as necessary.

For example:

```text
Start pin
    ↓
Perform native operation
    ↓
End pin
```

Long pinning periods can interfere with garbage collection and compaction.

---

# 6. How Pinning Affects Compaction

Suppose the heap contains:

```text
[A][B][C][D][E]
```

`B` is pinned and `C` becomes garbage:

```text
[A][B pinned][free][D][E]
```

The GC cannot simply move `B` during the pinning period.

Therefore the free space may remain fragmented.

---

# 7. Pinning and Fragmentation

A simplified example:

```text
Before:

[A][B][C][D][E]

B is pinned.
C becomes unreachable.

After collection:

[A][B pinned][free][D][E]
```

If `B` could move:

```text
[A][free][B][D][E]
```

the GC might be able to consolidate memory differently.

Because `B` is pinned, its movement is restricted.

Therefore:

> Excessive pinning can contribute to fragmentation.

---

# 8. `GCHandle` and Pinning

.NET also provides `GCHandle` for advanced scenarios.

Conceptually:

```csharp
GCHandle handle = GCHandle.Alloc(
    buffer,
    GCHandleType.Pinned);
```

The object remains pinned while the handle exists.

The handle must eventually be freed:

```csharp
handle.Free();
```

The important ownership rule is:

```text
Allocate pin
    ↓
Use pinned object
    ↓
Free pin
```

Forgetting to release such resources can cause problems.

---

# 9. Pinning Is Not the Same as Preventing Collection

This distinction is important.

Pinning means:

> The object cannot be moved.

It does not mean:

> The object can never be collected.

If an object is pinned but becomes unreachable, its exact behavior depends on the pinning mechanism and lifetime of the associated handle/root.

A `GCHandle` of type `Pinned` itself keeps a handle to the object, so the handle must be released when no longer needed.

The important conceptual distinction is:

```text
Pinning
   =
Cannot move

Reachability
   =
Can the object be collected?
```

These are different concepts.

---

# 10. Pinning and Native Interop

Pinning is particularly relevant when managed code communicates with native code.

Conceptually:

```text
Managed Code
     |
     v
Managed object
     |
     v
Pinned memory
     |
     v
Native code
```

The native side can safely use the stable memory address while the object is pinned.

---

# 11. Why Not Pin Everything?

Because pinning reduces the GC's freedom to compact memory.

If many objects are pinned:

```text
[Live][Pinned][Free][Pinned][Live][Free][Pinned]
```

the heap can become harder to compact efficiently.

Therefore:

> Pinning should be used deliberately and for the shortest practical duration.

---

# 12. Pinning Large Objects

Large objects already have different compaction behavior because they belong to the LOH.

Pinning can still matter for large objects when native code requires a stable address.

The broader principle remains:

```text
Pinned object
    ↓
GC cannot freely move it
```

---

# 13. Modern .NET and the Pinned Object Heap

Modern .NET also has a **Pinned Object Heap (POH)**.

The POH provides a dedicated area for certain pinned objects.

Conceptually:

```text
Managed Heap

SOH
LOH
POH
```

The POH helps the runtime manage pinned objects separately from movable objects.

This is an advanced runtime detail.

For interview preparation, remember:

> Modern .NET has a Pinned Object Heap designed to isolate certain pinned allocations and reduce their impact on other heap regions.

---

# 14. Pinning vs Copying

Sometimes instead of pinning a managed object, data can be copied into unmanaged memory or another buffer.

Conceptually:

```text
Option 1:

Managed object
     ↓
Pin
     ↓
Native code


Option 2:

Managed object
     ↓
Copy
     ↓
Unmanaged buffer
     ↓
Native code
```

The choice depends on the amount of data, lifetime, performance requirements, and API design.

---

# 15. Common Misconceptions

### Misconception 1

> "Pinned means the object cannot be collected."

Not necessarily.

Pinning primarily means the object cannot be moved while pinned.

---

### Misconception 2

> "Pinning is always bad."

False.

Pinning is a legitimate and necessary mechanism for some interop scenarios.

The problem is unnecessary or long-lived pinning.

---

### Misconception 3

> "The GC can move pinned objects during compaction."

No.

The purpose of pinning is to prevent movement during the relevant pinning period.

---

### Misconception 4

> "Pin every buffer before calling native code."

Not necessarily.

The correct approach depends on the interop API and how it handles memory.

---

# 16. Final Mental Model

```text
Normal object
      |
      v
GC can move it during compaction
      |
      v
Fragmentation can be reduced


Pinned object
      |
      v
GC cannot freely move it
      |
      v
Can interfere with compaction
      |
      v
Excessive pinning can contribute
to fragmentation
```

## One Sentence to Remember

> **Pinning temporarily prevents the GC from moving an object, which is useful for stable memory addresses during interop but can reduce compaction efficiency and contribute to fragmentation when overused.**
