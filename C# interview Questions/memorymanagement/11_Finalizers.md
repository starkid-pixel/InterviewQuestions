# C# Memory Management — Finalizers

## 1. Introduction

We have seen that `Dispose()` provides deterministic cleanup.

But what happens if a developer forgets to call `Dispose()`?

.NET provides another mechanism called a **finalizer**.

A finalizer allows a type to perform cleanup associated with unmanaged resources when the object is being processed by the GC.

---

# 2. What Is a Finalizer?

A finalizer is declared using destructor-like syntax:

```csharp
class ResourceHolder
{
    ~ResourceHolder()
    {
        // Cleanup
    }
}
```

In C#, this syntax represents a finalizer.

It is not the same as a deterministic destructor in languages such as C++.

---

# 3. Why Does a Finalizer Exist?

Suppose:

```text
Managed object
      |
      +----> unmanaged resource
```

The application forgets to call `Dispose()`.

Eventually the object may become unreachable:

```text
GC Roots

(no path)

ResourceHolder
```

A finalizer gives the runtime a mechanism to perform cleanup associated with the object before its memory is reclaimed.

---

# 4. Finalization Is Non-Deterministic

This is the most important point.

Calling:

```csharp
Dispose();
```

can happen at a known point.

A finalizer runs when the GC determines that the object is eligible for finalization and processes it.

Therefore:

```text
Dispose()
   ↓
Deterministic

Finalizer
   ↓
Non-deterministic
```

Do not use a finalizer when deterministic cleanup is required.

---

# 5. Finalizer and `Dispose()`

A type that owns unmanaged resources may use both:

```text
Dispose()
   ↓
Prompt cleanup

Finalizer
   ↓
Fallback cleanup
```

Conceptually:

```csharp
class ResourceHolder : IDisposable
{
    public void Dispose()
    {
        // Release resource
        GC.SuppressFinalize(this);
    }

    ~ResourceHolder()
    {
        // Fallback cleanup
    }
}
```

The exact implementation depends on the resource and modern .NET APIs.

---

# 6. Why `GC.SuppressFinalize()`?

If `Dispose()` has already performed the required cleanup, there is no reason for the object to go through finalization again.

Therefore:

```csharp
GC.SuppressFinalize(this);
```

tells the runtime that finalization is no longer required for this object.

Conceptually:

```text
Dispose()
   ↓
Resource released
   ↓
Suppress finalization
   ↓
No finalizer work required
```

---

# 7. Why Are Finalizers Expensive?

Finalizable objects require additional GC processing.

They cannot simply be treated like ordinary garbage immediately.

Conceptually:

```text
Object becomes unreachable
       ↓
Finalization processing
       ↓
Finalizer runs
       ↓
Object can later be reclaimed
```

This can extend the object's lifetime and add GC overhead.

Therefore:

> Avoid finalizers unless the type genuinely needs finalization semantics.

---

# 8. Finalization Can Delay Reclamation

A normal unreachable object can often be reclaimed during the relevant collection.

A finalizable object needs additional processing.

Conceptually:

```text
Normal object:

Unreachable
   ↓
Collect
   ↓
Memory reclaimed


Finalizable object:

Unreachable
   ↓
Finalization processing
   ↓
Finalizer runs
   ↓
Later collection
   ↓
Memory reclaimed
```

This is why finalizers should be used carefully.

---

# 9. Finalizers Are Not a Replacement for `Dispose()`

This is a critical interview point.

Incorrect:

```text
"I implemented a finalizer, so I don't need Dispose()."
```

A finalizer is not deterministic.

If a file handle must be released promptly, relying only on finalization is a poor design.

Preferred model:

```text
Consumer calls Dispose()
        ↓
Resource released promptly

Finalizer
        ↓
Fallback mechanism when appropriate
```

---

# 10. SafeHandle

Modern .NET provides `SafeHandle` to simplify unmanaged resource ownership.

Instead of manually implementing low-level finalization logic in every class, a resource-owning type can often use a `SafeHandle`.

Conceptually:

```text
Managed class
      |
      v
SafeHandle
      |
      v
Native/OS handle
```

`SafeHandle` incorporates reliable cleanup semantics and is generally preferred over writing raw finalizer-based handle management yourself.

---

# 11. Finalizers and Exceptions

Finalizers should not be designed to throw exceptions.

A finalizer runs under runtime-controlled cleanup processing, not normal application flow.

Resource cleanup code should therefore be defensive.

---

# 12. Common Misconceptions

### Misconception 1

> "A finalizer runs immediately when an object becomes unreachable."

False.

Finalization is non-deterministic.

---

### Misconception 2

> "Finalizer is the C# equivalent of a C++ destructor."

Not exactly.

C# finalizers are GC-driven and non-deterministic.

---

### Misconception 3

> "Every class should have a finalizer."

False.

Most classes should not.

---

### Misconception 4

> "Finalizers make Dispose unnecessary."

False.

`Dispose()` is the deterministic cleanup mechanism.

---

### Misconception 5

> "Finalizers are free."

False.

Finalization adds GC processing overhead.

---

# 13. Final Mental Model

```text
Object owns unmanaged resource
          |
          +-------------------+
          |                   |
      Dispose()           Finalizer
          |                   |
          v                   v
   Deterministic         Fallback
     cleanup             cleanup
          |                   |
          +---------+---------+
                    |
                    v
             Resource released
```

## One Sentence to Remember

> **A finalizer provides non-deterministic fallback cleanup for types that need finalization semantics, while `Dispose()` remains the preferred deterministic cleanup mechanism.**
