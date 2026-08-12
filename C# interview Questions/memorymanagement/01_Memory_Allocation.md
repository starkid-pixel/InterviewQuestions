# C# Memory Management — Memory Allocation

## Introduction

Before understanding garbage collection, first understand what memory is being allocated and what the variables represent.

```text
Variable
   ↓
Value OR Reference
   ↓
Storage
   ↓
Object lifetime
```

## Value Types

A value-type variable directly contains its value.

```csharp
int x = 10;
```

Conceptually:

```text
x
┌──────┐
│  10  │
└──────┘
```

Examples include `int`, `double`, `bool`, `struct`, and `enum`.

## Reference Types

A reference-type variable contains a reference to an object.

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

The variable and the object are different things.

## Important Rule

Do not use the simplified rule:

> "Value types are always on the stack and reference types are always on the heap."

That is not generally correct.

The actual location depends on the context in which the value or reference is stored.

For example, a value-type field is stored inline inside its containing object.

```csharp
class Person
{
    public int Age;
}
```

The `Age` value is part of the `Person` object.

Similarly, a reference-type field contains a reference inline inside its containing object:

```csharp
class Person
{
    public Address Address;
}
```

Conceptually:

```text
Person object
+----------------------+
| Age value            |
| Address reference ---|----> Address object
+----------------------+
```

## Key Mental Model

Think in terms of:

```text
VARIABLE
   ↓
What does it contain?
   ↓
Value OR Reference
   ↓
Where is that variable stored?
   ↓
If it is a reference, where is the referenced object stored?
```

This is more accurate than simply saying "value type = stack" and "reference type = heap".
