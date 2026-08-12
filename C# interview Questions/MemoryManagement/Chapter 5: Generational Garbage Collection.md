# Generational Garbage Collection

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
```

We know that when an object becomes unreachable, it becomes eligible for garbage collection.

Now an important question arises:

> **Does the .NET GC treat every object in exactly the same way?**

No.

The .NET Garbage Collector uses a **generational approach**.

The basic idea is:

> **Objects that have survived garbage collections are treated differently from newly created objects.**

The managed heap is divided into three main generations:

```text
Gen 0
Gen 1
Gen 2
```

---

# 2. Why Does Generational GC Exist?

Consider a typical application.

It creates many temporary objects:

```csharp
var message = CreateMessage();
var result = Process(message);
```

Some objects may exist only briefly.

Other objects may remain alive for a very long time:

```text
Application
    |
    +---- configuration
    +---- services
    +---- caches
    +---- long-lived objects
```

So applications commonly have:

```text
Many short-lived objects
        +
Fewer long-lived objects
```

The GC takes advantage of this behavior.

Instead of treating every object as equally likely to be garbage, it organizes objects into generations.

---

# 3. The Three Generations

The .NET GC has three primary generations:

```text
Gen 0
Gen 1
Gen 2
```

A simplified mental model is:

```text
New objects
     ↓
   Gen 0
     ↓
survives collection
     ↓
   Gen 1
     ↓
survives collection
     ↓
   Gen 2
```

The generation number does **not** mean:

> "Gen 2 objects are always better or more important."

It mainly represents how long objects have survived previous collections.

---

# 4. Gen 0 — Newly Created Objects

**Gen 0** contains newly allocated objects that are candidates for the first level of garbage collection.

For example:

```csharp
Person person = new Person();
```

Conceptually:

```text
new Person()
     ↓
Gen 0
```

Gen 0 is where many newly created short-lived objects are initially placed.

For example:

```text
Gen 0

Object A
Object B
Object C
Object D
Object E
```

Some of these objects may become unreachable very quickly.

---

# 5. Why Is Gen 0 Important?

Many objects in real applications are short-lived.

For example:

```csharp
for (int i = 0; i < 100000; i++)
{
    var item = CreateTemporaryObject();
}
```

A large number of temporary objects may be created.

Many of them become unreachable quickly.

If the GC had to examine the entire managed heap every time it wanted to clean up these temporary objects, that could be expensive.

Generational GC allows the runtime to focus collection work where it is often most useful.

Conceptually:

```text
Many new objects
       ↓
     Gen 0
       ↓
Many become unreachable
       ↓
Collect Gen 0
```

---

# 6. What Happens During a Gen 0 Collection?

Suppose Gen 0 contains:

```text
Gen 0

A
B
C
D
E
```

Assume:

```text
A → reachable
B → unreachable
C → unreachable
D → reachable
E → unreachable
```

A collection can reclaim the memory associated with:

```text
B
C
E
```

The surviving objects:

```text
A
D
```

continue to live.

Conceptually:

```text
Before:

[A][B][C][D][E]

After collection:

[A][D]
```

The actual runtime behavior is more sophisticated, but this gives us the basic idea.

---

# 7. What Happens to an Object That Survives?

This is where generations become important.

Suppose:

```text
Gen 0

Person object
```

The object survives a Gen 0 collection because it is still reachable.

It may then be **promoted** to the next generation:

```text
Gen 0
  |
  | survives collection
  v
Gen 1
```

So conceptually:

```text
New object
    ↓
  Gen 0
    ↓
survives GC
    ↓
  Gen 1
```

---

# 8. What Does Promotion Mean?

**Promotion** means that an object that survives a collection is moved to an older generation.

For example:

```text
Before:

Gen 0
  |
  +----> Person
```

After surviving a Gen 0 collection:

```text
Gen 1
  |
  +----> Person
```

The object has survived a collection and is therefore considered longer-lived.

---

# 9. Gen 1 — The Middle Generation

Gen 1 sits between Gen 0 and Gen 2.

Conceptually:

```text
Gen 0
  ↓
Gen 1
  ↓
Gen 2
```

Gen 1 serves an important role.

It separates:

```text
Very short-lived objects
```

from:

```text
Longer-lived objects
```

An object that survives Gen 0 can be promoted to Gen 1.

If it continues to survive collections, it can eventually be promoted to Gen 2.

---

# 10. Gen 2 — Long-Lived Objects

**Gen 2** contains objects that have survived enough collections to be considered long-lived.

Examples might include objects that remain alive for much of the application's lifetime:

```text
Application
    |
    +---- configuration
    +---- long-lived services
    +---- long-lived caches
```

Conceptually:

```text
Gen 2

Long-lived objects
```

Gen 2 is treated differently from Gen 0 because objects in Gen 2 are generally more likely to remain alive.

---

# 11. The Promotion Path

The simplified lifecycle is:

```text
Object created
      ↓
    Gen 0
      ↓
survives collection
      ↓
    Gen 1
      ↓
survives collection
      ↓
    Gen 2
```

This is the basic generational model.

However, not every object necessarily reaches Gen 2.

Many objects die in Gen 0.

For example:

```text
Object A
   ↓
Gen 0
   ↓
unreachable
   ↓
collected
```

It never needs to become Gen 1.

---

# 12. Why Is This More Efficient?

Suppose an application has:

```text
1,000,000 objects
```

but only:

```text
10,000 objects
```

are long-lived.

A generational strategy allows the runtime to focus more frequently on newer objects.

Conceptually:

```text
Gen 0
Many objects
Many temporary objects
       ↓
Frequent collection
```

while:

```text
Gen 2
Fewer objects
More long-lived objects
       ↓
Collected less frequently
```

This is based on an important observation:

> **Recently allocated objects are often more likely to become garbage than long-lived objects.**

---

# 13. Does Gen 2 Mean "Never Collected"?

No.

A Gen 2 object can still become unreachable.

For example:

```text
Gen 2
  |
  +----> Cache object
```

Later, if the application removes all references to that object:

```text
GC Roots

(no path)

Cache object
      X
```

The object can become eligible for collection.

So:

```text
Gen 2
  ≠
Never collected
```

Instead:

```text
Gen 2
  =
Longer-lived object that has survived
previous collections
```

---

# 14. A Complete Example

Consider:

```csharp
Person person = new Person();
```

Initially:

```text
Gen 0
  |
  +----> Person
```

Suppose the `Person` object remains reachable when a Gen 0 collection occurs.

It may be promoted:

```text
Gen 1
  |
  +----> Person
```

Suppose it continues to survive later collections.

It may eventually become:

```text
Gen 2
  |
  +----> Person
```

If the application later removes the last reference:

```text
GC Roots

(no path)

Person
```

the object becomes unreachable and can eventually be collected.

---

# 15. What Happens to Short-Lived Objects?

Consider:

```csharp
void Process()
{
    var data = new TemporaryData();
}
```

The object may follow a simple path:

```text
Create
  ↓
Gen 0
  ↓
Method finishes
  ↓
Object becomes unreachable
  ↓
Gen 0 collection
  ↓
Memory reclaimed
```

The object may never be promoted.

This is exactly the type of behavior generational GC is designed to handle efficiently.

---

# 16. What Happens to Long-Lived Objects?

Consider an object that remains referenced for a long time:

```text
Application
   |
   v
Long-lived object
```

Conceptually:

```text
Gen 0
   ↓
survives
   ↓
Gen 1
   ↓
survives
   ↓
Gen 2
```

It has demonstrated that it tends to remain alive.

The runtime can therefore treat it as a long-lived object.

---

# 17. Are Generations Separate Heaps?

It is better to think of generations as **logical regions/categories within the managed heap**, rather than three completely independent heaps.

Conceptually:

```text
Managed Heap
+--------------------------------------+
| Gen 0 | Gen 1 | Gen 2 |     ...      |
+--------------------------------------+
```

The actual .NET GC implementation is more sophisticated, and the physical organization can vary depending on runtime behavior and GC mode.

For learning purposes, remember:

> **Gen 0, Gen 1, and Gen 2 are generations managed by the GC to optimize collection based on object lifetime.**

---

# 18. Does Every Object Move from Gen 0 to Gen 1 to Gen 2?

No.

An object can become unreachable at any point.

For example:

```text
Object created
      ↓
Gen 0
      ↓
becomes unreachable
      ↓
collected
```

It never reaches Gen 1.

Another object might:

```text
Gen 0
  ↓
Gen 1
  ↓
Gen 2
```

The object's path depends on how long it remains reachable and how collections occur.

---

# 19. Important Misconception: Generation Is Not Age in Seconds

A generation does not simply mean:

```text
Gen 0 = young for 10 seconds
Gen 1 = 20 seconds old
Gen 2 = 30 seconds old
```

That is not the model.

Instead, generations are related to **survival across garbage collections**.

Conceptually:

```text
Created
   ↓
Gen 0
   ↓
survives collection
   ↓
older generation
```

So generation is better understood as:

> **A classification based largely on survival through collections.**

---

# 20. What Does "Short-Lived" Mean?

Short-lived does not mean:

> "The object exists for exactly a certain number of milliseconds."

It means:

> **The object tends to become unreachable relatively quickly.**

For example:

```csharp
var temporary = new TemporaryObject();
```

If the reference disappears soon afterward:

```text
Create
 ↓
Use
 ↓
Reference disappears
 ↓
Object becomes unreachable
```

it is a short-lived object from the GC's perspective.

---

# 21. What Does "Long-Lived" Mean?

A long-lived object is one that remains reachable across multiple garbage collections.

For example:

```text
Application
   |
   v
Configuration object
```

If the configuration object remains reachable for a long time, it may survive multiple collections and eventually be promoted to Gen 2.

Therefore:

```text
Long-lived
    =
Survives multiple collections
```

---

# 22. A Simple Visual Model

Think of the generations like this:

```text
                NEW OBJECTS
                     |
                     v
                  Gen 0
                     |
             survives collection
                     |
                     v
                  Gen 1
                     |
             survives collection
                     |
                     v
                  Gen 2
                     |
              long-lived objects
```

At any stage:

```text
Object becomes unreachable
          ↓
Eligible for collection
```

So an object does not have to reach Gen 2 before it can be collected.

---

# 23. How This Connects to GC

We can now combine the previous chapters.

### Step 1 — Allocation

```text
new Person()
```

creates an object.

### Step 2 — Gen 0

```text
Person
  ↓
Gen 0
```

### Step 3 — Reachability

If a GC root can reach the object:

```text
GC Root
   |
   v
Person
```

it remains alive.

### Step 4 — Collection

If the object becomes unreachable:

```text
GC Root

(no path)

Person
```

it becomes eligible for collection.

### Step 5 — Promotion

If it survives a collection while remaining reachable:

```text
Gen 0
  ↓
Gen 1
```

and potentially:

```text
Gen 1
  ↓
Gen 2
```

This gives us a complete conceptual model.

---

# 24. Why Gen 0 Collections Can Be Frequent

Many applications create temporary objects.

For example:

```text
Request
   ↓
Create temporary objects
   ↓
Process data
   ↓
Objects become unreachable
```

These objects are often in Gen 0.

Therefore the runtime can frequently collect Gen 0 to reclaim memory from short-lived objects.

This is generally more efficient than repeatedly processing the entire population of long-lived objects.

---

# 25. Why Gen 2 Collections Are More Significant

Gen 2 contains longer-lived objects.

A Gen 2 collection can involve more objects and more work than a small Gen 0 collection.

Therefore, in simplified terms:

```text
Gen 0 collection
    ↓
Usually focused on newer objects

Gen 2 collection
    ↓
Deals with older, long-lived objects
```

The real runtime has additional behavior and optimizations, but this distinction is important for understanding GC performance.

---

# 26. Common Misconceptions

### Misconception 1

> "Every object eventually reaches Gen 2."

False.

Many objects become unreachable and are collected while still in Gen 0.

---

### Misconception 2

> "Gen 2 objects cannot be collected."

False.

If a Gen 2 object becomes unreachable, it can eventually be collected.

---

### Misconception 3

> "Generation means the object's exact age."

False.

Generation is related to survival across collections, not simply elapsed time.

---

### Misconception 4

> "Gen 0 contains objects that are exactly zero years old."

False.

Gen 0 represents the newest generation of managed objects, not an exact age.

---

### Misconception 5

> "Gen 0, Gen 1, and Gen 2 are completely independent heaps."

That is an oversimplification.

They are generations managed as part of the managed heap.

---

# 27. Final Mental Model

The most useful model is:

```text
                  OBJECT CREATED
                        |
                        v
                     GEN 0
                        |
             +----------+----------+
             |                     |
       becomes unreachable     survives GC
             |                     |
             v                     v
         COLLECTED               GEN 1
                                       |
                              +--------+--------+
                              |                 |
                        becomes unreachable  survives GC
                              |                 |
                              v                 v
                          COLLECTED           GEN 2
                                                  |
                                         remains reachable
                                                  |
                                                  v
                                           Long-lived object
```

The central idea is:

> **Generational GC takes advantage of the fact that many objects die young, while objects that survive multiple collections are more likely to remain alive longer.**

---

# 28. One Sentence to Remember

> **.NET uses generations to make garbage collection more efficient by treating newly allocated, short-lived objects differently from objects that have survived previous collections and become long-lived.**

The next topic is:

```text
Generational GC
      ↓
Large Object Heap (LOH)
```

The LOH is important because large objects are handled differently from ordinary small object allocations and introduces another important part of .NET memory management.
