# C# Memory Management — Object Lifetime

## Introduction

An object's lifetime is the period during which the object exists in managed memory.

Consider:

```csharp
Person p = new Person();
```

Conceptually:

```text
p
 |
 v
Person object
```

The object remains alive while it is reachable.

## Lifetime Is About Reachability

The important question for the GC is not:

> "Is this variable still in scope?"

The important question is:

> "Can the object still be reached from a GC root?"

An object can become unreachable before a surrounding method finishes.

Conversely, an object can remain alive because another reference still points to it.

## Example

```csharp
Person p = new Person();

Person another = p;

p = null;
```

The `Person` object is still reachable through `another`.

```text
another
   |
   v
Person object
```

Therefore it is not garbage.

If both references disappear:

```text
GC Roots

(no path)

Person object
```

the object becomes eligible for garbage collection.

## Key Rule

> An object becomes eligible for collection when it is no longer reachable from any GC root.
