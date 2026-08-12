# C# Memory Management — GC Roots and Reachability

## What Is a GC Root?

A GC root is a starting point from which the garbage collector determines which managed objects are still reachable.

Conceptually:

```text
GC Root
   |
   v
Object A
   |
   v
Object B
```

Because `B` can be reached from a root, `B` is considered live.

## Common Root Categories

The runtime can maintain references through mechanisms such as:

- Active execution state and references held by running code
- Static references
- Handles maintained by the runtime
- Other runtime-managed roots

The exact implementation is runtime-specific.

## Reachability

Consider:

```csharp
Person p = new Person();
```

Conceptually:

```text
GC Root
   |
   v
p
   |
   v
Person
```

The object is reachable.

Now:

```csharp
p = null;
```

Assuming no other references exist:

```text
GC Root

(no path)

Person
```

The object is unreachable and eligible for collection.

## Object Graph

Objects can reference other objects:

```text
Root
 |
 v
A ---> B ---> C
 |
 +---> D
```

All of these are reachable.

If the root loses its reference to `A`, the entire disconnected graph may become eligible for collection.

## Key Rule

> The GC is primarily concerned with reachability, not simply variable scope.
