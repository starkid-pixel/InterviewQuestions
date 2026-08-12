# C#/.NET Generational Garbage Collection — Concept Document

## 1. The Three Generations

.NET uses three main generations:

- Gen 0 — newly allocated, usually short-lived objects
- Gen 1 — intermediate generation
- Gen 2 — long-lived objects

The simplified object lifetime is:

```text
New Object
    ↓
  Gen 0
    ↓ survives collection
  Gen 1
    ↓ survives collection
  Gen 2
```

There is no normal Gen 3.

---

## 2. Why Generations Exist

Most objects have short lifetimes. It is therefore inefficient to examine the entire managed heap every time garbage collection is needed.

The GC can collect younger generations more frequently:

```text
Gen 0 → collected frequently
Gen 1 → collected less frequently
Gen 2 → collected least frequently
```

---

## 3. New Objects Start in Gen 0

A normal managed allocation such as:

```csharp
var customer = new Customer();
```

normally begins its generational lifetime in Gen 0.

If the object remains reachable and survives collections, it may be promoted to an older generation.

---

## 4. Object Promotion

The simplified model is:

```text
Gen 0
  │
  │ survives
  ▼
Gen 1
  │
  │ survives
  ▼
Gen 2
```

Do not interpret this as:

> Every GC moves every Gen 0 object to Gen 1 and every Gen 1 object to Gen 2.

Only surviving objects can be promoted, and the runtime controls the actual promotion behavior.

---

# 5. The Most Important Concept: Collection Scope

The generation number of a collection describes its scope.

### Gen 0 collection

```text
Gen 0 GC
    ↓
Gen 0
```

### Gen 1 collection

```text
Gen 1 GC
    ↓
Gen 0 + Gen 1
```

### Gen 2 collection

```text
Gen 2 GC
    ↓
Gen 0 + Gen 1 + Gen 2
```

Therefore:

| Collection | Generations in scope |
|---|---|
| Gen 0 GC | Gen 0 |
| Gen 1 GC | Gen 0 + Gen 1 |
| Gen 2 GC | Gen 0 + Gen 1 + Gen 2 |

This distinction is extremely important for interviews.

---

# 6. Does Every GC Go G0 → G1 → G2?

No.

Do not think:

```text
Gen 0 GC
   ↓
Gen 1 GC
   ↓
Gen 2 GC
```

for every collection.

The runtime determines the appropriate collection scope using factors such as:

- Allocation pressure
- Available memory
- Generation thresholds
- GC heuristics
- Runtime conditions
- Memory pressure

A collection may be a Gen 0, Gen 1, or Gen 2 collection.

---

# 7. Can the Runtime Go Directly to Gen 2?

Yes.

A Gen 2 collection can occur without first performing two separate GC operations:

```text
Gen 0 GC
   ↓
Gen 1 GC
   ↓
Gen 2 GC
```

Instead, the runtime can perform:

```text
Gen 2 GC
   ↓
Gen 0 + Gen 1 + Gen 2
```

A Gen 2 collection is therefore a broader collection, not necessarily the third step of a visible sequence of three separate collections.

---

# 8. What Happens During a Gen 0 Collection?

Suppose:

```text
Gen 0:
A B C D E
```

Assume:

```text
A → reachable
B → unreachable
C → reachable
D → unreachable
E → reachable
```

The unreachable objects can be reclaimed.

The survivors remain live and may be promoted:

```text
A C E
 ↓ ↓ ↓
may be promoted
```

New allocations continue to go into Gen 0.

---

# 9. What Happens During a Gen 1 Collection?

Suppose:

```text
Gen 0:
A B C

Gen 1:
D E F

Gen 2:
G H
```

A Gen 1 collection considers:

```text
Gen 0 + Gen 1
```

Gen 2 is not part of that generational collection.

Unreachable objects in the relevant generations can be reclaimed.

Surviving objects can remain in their generation or be promoted according to runtime behavior.

---

# 10. What Happens During a Gen 2 Collection?

Suppose:

```text
Gen 0:
A B C

Gen 1:
D E F

Gen 2:
G H I
```

A Gen 2 collection has:

```text
Gen 0 + Gen 1 + Gen 2
```

in scope.

Unreachable objects from the relevant generations can be reclaimed.

Objects in Gen 2 that survive remain in Gen 2.

There is no normal Gen 3.

---

# 11. Does Gen 2 Being Full Mean Immediate Collection?

No.

Avoid the simplistic rule:

```text
Gen 2 full
    ↓
Immediately collect Gen 2
```

The runtime uses allocation pressure, memory conditions, thresholds, GC history, and heuristics to determine when collection should occur.

Gen 2 collections are generally more expensive, so the runtime tries to avoid unnecessary full collections.

---

# 12. Collection vs Promotion

These are different concepts.

### Collection

Collection determines which objects are unreachable and reclaims their memory.

```text
Object
  ↓
Unreachable
  ↓
Eligible for collection
  ↓
Memory reclaimed
```

### Promotion

Promotion means a surviving object may move to an older generation:

```text
Gen 0
  ↓ survives
Gen 1
```

or:

```text
Gen 1
  ↓ survives
Gen 2
```

Do not confuse promotion with collection.

---

# 13. Does Gen 0 Get Considered Again During a Gen 1 Collection?

Yes.

A Gen 1 collection includes Gen 0 and Gen 1:

```text
Gen 1 collection
      ↓
Gen 0 + Gen 1
```

Conceptually, Gen 0 is therefore considered again as part of the broader collection.

The actual GC implementation uses optimizations, so this should not be interpreted as necessarily performing two identical full scans of Gen 0.

---

# 14. Can Gen 0 Be Collected Without Gen 1?

Yes.

This is one of the main benefits of generational GC.

```text
Gen 0 GC
   ↓
Gen 0 considered
```

Gen 1 and Gen 2 do not need to be collected every time.

---

# 15. Can Gen 1 Be Collected Without a Separate Gen 0 Collection Immediately Before It?

Yes.

A Gen 1 collection itself includes Gen 0:

```text
Gen 1 GC
   ↓
Gen 0 + Gen 1
```

There does not have to be a separate preceding Gen 0 GC operation.

---

# 16. Can Gen 2 Be Collected Directly?

Yes.

A Gen 2 collection includes:

```text
Gen 0 + Gen 1 + Gen 2
```

It does not require three separately executed operations:

```text
G0 GC
G1 GC
G2 GC
```

beforehand.

---

# 17. Complete Mental Model

The safest mental model is:

```text
                 NEW OBJECT
                     │
                     ▼
                  GEN 0
                     │
             survives collection
                     │
                     ▼
                  GEN 1
                     │
             survives collection
                     │
                     ▼
                  GEN 2
                     │
             survives collection
                     │
                     ▼
                STILL GEN 2
```

Collection scope:

```text
Gen 0 GC → G0

Gen 1 GC → G0 + G1

Gen 2 GC → G0 + G1 + G2
```

The key principle is:

> **Generation describes the age/lifetime category of an object, while the generation number of a collection describes the scope of that collection.**

---

# 18. Interview-Ready Answer

### Question

**How does generational GC work in .NET?**

### Answer

> .NET uses a generational garbage collector with Gen 0, Gen 1, and Gen 2. Newly allocated objects normally start in Gen 0. Objects that survive collections may be promoted to older generations, eventually reaching Gen 2.
>
> The GC does not necessarily execute every collection as Gen 0, then Gen 1, then Gen 2. The runtime determines an appropriate collection scope based on allocation pressure, memory conditions, thresholds, and GC heuristics.
>
> A Gen 0 collection considers Gen 0, a Gen 1 collection considers Gen 0 and Gen 1, and a Gen 2 collection considers Gen 0, Gen 1, and Gen 2.
>
> Gen 2 is the oldest generation. Objects that survive a Gen 2 collection remain in Gen 2 because there is no normal Gen 3.

---

# 19. Common Interview Mistakes

### Mistake 1

> Every GC goes from Gen 0 to Gen 1 to Gen 2.

**Correction:** A collection can target Gen 0, Gen 1, or Gen 2 depending on runtime conditions.

### Mistake 2

> A Gen 1 GC only collects Gen 1.

**Correction:** A Gen 1 collection includes Gen 0 and Gen 1.

### Mistake 3

> A Gen 2 GC only collects Gen 2.

**Correction:** A Gen 2 collection includes Gen 0, Gen 1, and Gen 2.

### Mistake 4

> Gen 2 being full means GC immediately runs.

**Correction:** The runtime uses allocation pressure, memory conditions, thresholds, and GC heuristics.

### Mistake 5

> All Gen 0 objects move to Gen 1 after every GC.

**Correction:** Only surviving objects can be promoted, and promotion behavior is controlled by the runtime.

### Mistake 6

> Gen 2 objects move to Gen 3 when they survive.

**Correction:** There is no normal Gen 3. Surviving Gen 2 objects remain Gen 2.

---

# 20. Final Summary

Remember these three rules:

### Rule 1 — New objects start in Gen 0

```text
New object
    ↓
Gen 0
```

### Rule 2 — Survivors may be promoted

```text
Gen 0 → Gen 1 → Gen 2
```

### Rule 3 — Higher collection generation means broader scope

```text
Gen 0 GC → G0

Gen 1 GC → G0 + G1

Gen 2 GC → G0 + G1 + G2
```

The central concept is:

> **The GC does not have to progress through G0, G1, and G2 as three separate operations. It can perform a collection at the required generation. The higher the collection generation, the broader the collection scope.**
