# C# Memory Management — Generational Garbage Collection

## 1. Introduction

The .NET Garbage Collector uses a generational approach.

The basic idea is:

> Objects that have survived garbage collections are treated differently from newly created objects.

The managed heap has three primary generations:

```text
Gen 0
Gen 1
Gen 2
```

## 2. Why Generational GC Exists

Applications commonly create many short-lived objects and fewer long-lived objects.

```text
Many short-lived objects
        +
Fewer long-lived objects
```

The GC takes advantage of this behavior.

## 3. The Three Generations

A simplified model is:

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

Generation should not be interpreted simply as elapsed age.

It is related to survival across collections.

## 4. Gen 0

Newly allocated objects generally start in Gen 0.

```csharp
Person person = new Person();
```

Conceptually:

```text
new Person()
     ↓
Gen 0
```

Many short-lived objects become unreachable while still in Gen 0.

## 5. Gen 0 Collection

Suppose:

```text
A → reachable
B → unreachable
C → unreachable
D → reachable
E → unreachable
```

A collection can reclaim the memory associated with `B`, `C`, and `E`.

The surviving objects remain alive.

## 6. Promotion

If an object survives a collection, it may be promoted to an older generation.

```text
Gen 0
  |
  | survives collection
  v
Gen 1
```

A longer-lived object may eventually reach Gen 2.

## 7. Gen 1

Gen 1 sits between Gen 0 and Gen 2.

```text
Gen 0
  ↓
Gen 1
  ↓
Gen 2
```

It provides an intermediate generation between very short-lived and longer-lived objects.

## 8. Gen 2

Gen 2 contains objects that have survived enough collections to be considered long-lived.

Examples may include long-lived configuration, services, and caches.

## 9. Does Gen 2 Mean Never Collected?

No.

A Gen 2 object can become unreachable and can eventually be collected.

```text
Gen 2
  |
  +----> Object
```

If the last reference disappears, the object becomes eligible for collection.

## 10. Does Every Object Reach Gen 2?

No.

An object may become unreachable in Gen 0:

```text
Created
  ↓
Gen 0
  ↓
unreachable
  ↓
collected
```

## 11. Generation Is Not Exact Age

This is incorrect:

```text
Gen 0 = 10 seconds old
Gen 1 = 20 seconds old
Gen 2 = 30 seconds old
```

Instead:

```text
Created
   ↓
Gen 0
   ↓
survives collection
   ↓
older generation
```

## 12. Final Mental Model

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

## One Sentence to Remember

> .NET uses generations to make garbage collection more efficient by treating newly allocated, short-lived objects differently from objects that have survived previous collections and become long-lived.
