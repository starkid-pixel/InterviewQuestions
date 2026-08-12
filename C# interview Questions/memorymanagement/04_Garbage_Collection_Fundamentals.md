# C# Memory Management — Garbage Collection Fundamentals

## What Is Garbage Collection?

Garbage collection is the .NET runtime's mechanism for automatically reclaiming memory occupied by managed objects that are no longer reachable.

Conceptually:

```text
Object created
     ↓
Object used
     ↓
Object becomes unreachable
     ↓
GC identifies it
     ↓
Memory can be reclaimed
```

## Why Does .NET Need GC?

Without automatic memory management, developers would have to manually track the lifetime of every managed object.

The GC provides automatic reclamation for managed memory.

## Important Distinction

When an object becomes unreachable, it becomes:

> **Eligible for garbage collection**

That does not mean:

> **It is immediately destroyed.**

The GC decides when collection occurs.

## Example

```csharp
void Process()
{
    var person = new Person();
}
```

After the method finishes, assuming no other references exist:

```text
GC Roots

(no path)

Person
```

The object is eligible for collection.

## GC Does Not Collect Reachable Objects

If an object remains reachable:

```text
GC Root
   |
   v
Person
```

the GC must preserve it.

## Key Rule

> Garbage collection reclaims memory for unreachable managed objects; becoming unreachable and being collected are two different events.
