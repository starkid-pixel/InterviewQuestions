Chapter 3: # GC Roots and Reachability

## 1. Introduction

In the previous chapter, we established an important rule:

> An object does not become collectible simply because a variable goes out of scope. The object becomes eligible for garbage collection when it is no longer reachable from a GC root.

That raises the next question:

> **What is a GC root, and how does the GC determine whether an object is reachable?**

To answer this, we need to understand the relationship between:

- GC roots
- references
- objects
- object graphs
- reachability

---

# 2. What Is a GC Root?

A **GC root** is a reference that the garbage collector treats as a starting point when determining which managed objects are still reachable.

Think of a GC root as an **entry point into the object graph**.

For example:

```text
GC Root
   |
   v
Object A
```

The GC starts from the root and follows references.

If `Object A` contains a reference to `Object B`:

```text
GC Root
   |
   v
Object A
   |
   v
Object B
```

then `Object B` is also reachable.

The important idea is:

> **The GC starts from roots and follows references to determine which objects are reachable.**

---

# 3. Why Does the GC Need Roots?

Suppose the managed heap contains:

```text
Object A
Object B
Object C
Object D
Object E
```

The GC needs to determine:

> Which of these objects are still being used?

It cannot simply ask whether an object exists.

Instead, it asks:

> **Can this object be reached from a GC root?**

For example:

```text
GC Roots
   |
   +------> Object A
   |           |
   |           +------> Object B
   |
   +------> Object C
```

Here:

```text
Object A → reachable
Object B → reachable
Object C → reachable
```

If `Object D` and `Object E` cannot be reached from any root:

```text
GC Roots
   |
   +------> Object A
               |
               +------> Object B


Object D

Object E
```

then `Object D` and `Object E` are unreachable.

They can become eligible for garbage collection.

---

# 4. How Does Reachability Work?

Consider:

```csharp
Person person = new Person();
```

Conceptually:

```text
GC Root
   |
   v
person
   |
   v
Person object
```

The reference held by the active code provides a path to the `Person` object.

Therefore:

```text
GC Root
   ↓
Person object
```

The `Person` object is reachable.

---

# 5. Reachability Can Continue Through Multiple Objects

Consider:

```csharp
class Person
{
    public Address Address;
}

class Address
{
    public City City;
}
```

And:

```csharp
Person person = new Person();
person.Address = new Address();
person.Address.City = new City();
```

Conceptually:

```text
GC Root
   |
   v
Person
   |
   | Address
   v
Address
   |
   | City
   v
City
```

The GC can follow the chain:

```text
GC Root
   ↓
Person
   ↓
Address
   ↓
City
```

Therefore all three objects are reachable.

This is why we talk about an **object graph**.

---

# 6. What Is an Object Graph?

Objects can reference other objects.

Those relationships form a graph.

For example:

```text
GC Root
   |
   v
Person
  /  \
 v    v
Address Account
  |
  v
City
```

The graph represents:

```text
Person → Address
Person → Account
Address → City
```

The GC uses these reference relationships when determining reachability.

The important question is not:

> "How many objects exist?"

It is:

> **"Which objects can be reached from a GC root?"**

---

# 7. What Happens When the Root Reference Disappears?

Consider:

```csharp
void Test()
{
    Person p = new Person();
}
```

While `Test()` is executing:

```text
GC Root
   |
   v
p
   |
   v
Person object
```

The `Person` object is reachable.

When the method finishes, assume there are no other references to the object.

Now there is no path from a GC root:

```text
GC Roots

(no path)

Person object
      X
  unreachable
```

The object becomes **eligible for garbage collection**.

Remember:

> **Eligible for collection does not mean immediately collected.**

---

# 8. One Reference Can Keep an Object Alive

Consider:

```csharp
Person p1 = new Person();
Person p2 = p1;
```

There is only **one `Person` object**.

There are two references to it:

```text
p1 --------+
           |
           v
       Person object
           ^
           |
p2 --------+
```

Now:

```csharp
p1 = null;
```

We have:

```text
p1 → null

p2
 |
 v
Person object
```

The object is still reachable through `p2`.

Therefore:

> **Removing one reference does not necessarily make an object collectible.**

---

# 9. The Last Reachable Reference Matters

Continue the previous example:

```csharp
p2 = null;
```

Now:

```text
p1 → null
p2 → null

Person object
      X
  unreachable
```

If there are no other paths from GC roots, the object becomes eligible for garbage collection.

So the useful mental model is:

```text
References exist
      ↓
Object may be reachable
      ↓
References disappear
      ↓
No path from any GC root
      ↓
Object becomes unreachable
      ↓
Eligible for GC
```

---

# 10. Can One Object Keep Another Object Alive?

Yes.

Consider:

```csharp
class Person
{
    public Address Address;
}
```

and:

```csharp
Person person = new Person();
person.Address = new Address();
```

Conceptually:

```text
GC Root
   |
   v
Person
   |
   v
Address
```

The `Person` object keeps the `Address` object reachable because it contains a reference to it.

If the root still reaches `Person`, the GC can also reach `Address`.

Therefore both are reachable.

---

# 11. What Happens When the Root Is Removed?

Suppose:

```csharp
person = null;
```

and there are no other references.

Now:

```text
GC Roots

(no path)

Person
  |
  v
Address
```

The entire connected group is unreachable from the roots.

Therefore both objects can become eligible for collection.

This is an important property of tracing garbage collection:

> **The GC considers the entire reachable object graph, not individual variables in isolation.**

---

# 12. Circular References

Consider:

```text
Object A
   |
   v
Object B
   |
   +------> Object A
```

`Object A` references `Object B`.

`Object B` references `Object A`.

This is a circular reference.

A common misconception is:

> "Because they reference each other, they can never be collected."

That is incorrect.

The important question is:

> **Is the circular group reachable from a GC root?**

## If a GC Root Can Reach Them

```text
GC Root
   |
   v
Object A ------> Object B
   ^                 |
   |_________________|
```

They are reachable.

They remain alive.

## If No GC Root Can Reach Them

```text
GC Roots

(no path)

Object A ------> Object B
   ^                 |
   |_________________|
```

Even though the objects reference each other, the entire cycle is unreachable.

Therefore they can become eligible for garbage collection.

The important rule is:

> **A reference cycle by itself does not keep objects alive.**

---

# 13. Static References

Static references can be important because they can remain reachable for a long time.

For example:

```csharp
class Cache
{
    public static Person CurrentPerson;
}
```

Suppose:

```csharp
Cache.CurrentPerson = new Person();
```

Conceptually:

```text
GC Root
   |
   v
Static reference
   |
   v
Person object
```

As long as the static reference remains reachable, the `Person` object can remain reachable as well.

This is why long-lived static references should be used carefully.

The important principle is:

> **An object referenced by a long-lived root can remain alive for a long time.**

---

# 14. Local Variables and Reachability

Consider:

```csharp
void Test()
{
    Person p = new Person();

    // use p
}
```

During execution, the object can be reached through the active local/reference.

Conceptually:

```text
During execution:

GC Root
   |
   v
p
   |
   v
Person
```

Later:

```text
GC Roots

(no path)

Person
   X
unreachable
```

The exact point at which a local stops being considered live is a runtime/JIT implementation detail.

Therefore, avoid thinking:

> "The variable exists until the closing brace, so the object definitely remains alive until then."

The GC and JIT work with **actual liveness and reachability**, not simply source-code scope.

---

# 15. Scope vs Reachability

This distinction is extremely important.

### Scope

Scope answers:

> **Where can I refer to this variable in my source code?**

### Reachability

Reachability answers:

> **Can the GC still reach this object from a GC root?**

Therefore:

```text
Scope
  ≠
Reachability
```

And:

```text
Reachability
  ↓
helps determine object lifetime
```

This is why an object's lifetime should not be explained simply as:

```text
variable goes out of scope
        ↓
object destroyed
```

That model is incorrect.

---

# 16. A Complete Example

Consider:

```csharp
class Person
{
    public Address Address;
}

class Address
{
    public string City;
}
```

Then:

```csharp
Person p = new Person();
p.Address = new Address();
p.Address.City = "Chennai";
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
   |
   | Address
   v
Address
   |
   | City
   v
String
```

The GC can follow the reference chain.

Therefore the objects are reachable.

If `p` is removed and there are no other references:

```text
GC Roots

(no path)

Person
   |
   v
Address
   |
   v
String
```

The entire group is unreachable.

The objects can become eligible for collection.

---

# 17. The Core Rule

Everything in this chapter comes down to one question:

> **Can the object be reached from a GC root by following references?**

If yes:

```text
GC Root
   |
   v
Object
```

The object is reachable.

If no:

```text
GC Root

(no path)

Object
```

The object is unreachable and can become eligible for garbage collection.

---

# 18. Final Mental Model

Keep this model in mind:

```text
                    GC ROOTS
                       |
             +---------+---------+
             |                   |
             v                   v
         Object A            Object C
             |
       +-----+-----+
       |           |
       v           v
   Object B    Object D
       |
       v
   Object E
```

The GC starts at the roots and follows references.

Therefore:

```text
GC Root
   ↓
reachable object
   ↓
reachable object
   ↓
reachable object
```

All of those objects are reachable.

Anything for which there is **no path from any GC root** is unreachable:

```text
GC ROOTS

(no path)

Object X
   |
   v
Object Y
```

Even though `X` references `Y`, neither is reachable from a root.

Both can become eligible for collection.

---

# 19. One Sentence to Remember

> **GC roots are the starting points used by the garbage collector to determine reachability; an object remains alive from the GC's perspective as long as a path of references exists from a GC root to that object.**

This is the foundation for understanding **garbage collection, generations, memory leaks, static references, and object lifetime**.
