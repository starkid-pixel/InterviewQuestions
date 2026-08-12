# Object Lifetime — How Long Does an Object Live?

## 1. Introduction

The previous chapter established an important distinction:

```text
Value type
    → actual value

Reference type
    → reference to an object
```

We also learned that a reference variable and the object it refers to are **two different things**.

For example:

```csharp
Person p = new Person();
```

Conceptually:

```text
p
 |
 | reference
 v
Person object
```

The next question is:

> **How long does the `Person` object live?**

This is the starting point for understanding **garbage collection and object lifetime in .NET**.

---

# 2. Object Creation Does Not Mean Immediate Destruction

Consider:

```csharp
void Test()
{
    Person p = new Person();
}
```

When this line executes:

```csharp
Person p = new Person();
```

a `Person` object is created.

While `p` refers to it:

```text
p
 |
 v
Person object
```

When the method finishes, the local variable `p` is no longer available to the method.

A common misconception is:

> "The method ended, so the `Person` object is immediately destroyed."

That is **not how managed object lifetime works**.

The object becomes eligible for garbage collection only when it is no longer reachable through any reference that the GC considers a root.

So:

```text
Method starts
     ↓
Person object created
     ↓
p refers to Person
     ↓
Method ends
     ↓
p is no longer a usable local
     ↓
Is the object reachable anywhere else?
     ↓
No
     ↓
Object becomes eligible for GC
```

Notice the wording:

> **Eligible for GC**

not:

> **Immediately destroyed**

---

# 3. Scope and Object Lifetime Are Not the Same Thing

This is one of the most important concepts.

Consider:

```csharp
void Test()
{
    Person p = new Person();
}
```

The **scope** of `p` ends when the method ends.

But the **lifetime of the object** is determined by reachability and garbage collection.

Therefore:

```text
Variable scope
      ≠
Object lifetime
```

The variable can disappear while the object remains alive.

---

# 4. An Object Can Outlive a Local Variable

Consider:

```csharp
Person CreatePerson()
{
    Person p = new Person();
    return p;
}
```

The local variable `p` belongs to the method.

When the method returns, `p` is gone.

But the object is still reachable because the reference has been returned:

```csharp
Person person = CreatePerson();
```

Conceptually:

```text
CreatePerson()

p
 |
 v
Person object
 |
 | returned reference
 v

person
 |
 v
same Person object
```

So the object continues to live after the local variable `p` has disappeared.

This demonstrates:

> **The lifetime of an object is not tied to the lifetime of one particular variable.**

What matters is whether the object is still reachable.

---

# 5. What Does Reachable Mean?

An object is **reachable** when the garbage collector can follow references from a GC root to that object.

For example:

```text
GC Root
   |
   v
Person A
   |
   v
Address
```

The `Address` object is reachable because the GC can start from the root and follow the reference to `Person A`, and then from `Person A` to `Address`.

Conceptually:

```text
GC Root
   |
   +----> Person
             |
             +----> Address
```

As long as that path exists, the object is considered reachable.

---

# 6. What Is a GC Root?

A **GC root** is a reference that the garbage collector treats as a starting point when determining which objects are still reachable.

Typical examples include:

- active references held by running code
- static references
- references maintained by certain runtime mechanisms
- other runtime-managed roots

The important idea is not memorizing every type of root.

The important idea is:

> **The GC starts from roots and follows references.**

Conceptually:

```text
GC ROOTS
   |
   +------> Object A
   |           |
   |           +------> Object B
   |
   +------> Object C
               |
               +------> Object D
```

Objects reachable from these roots are considered alive from the GC's perspective.

---

# 7. What Happens When an Object Is No Longer Reachable?

Consider:

```csharp
void Test()
{
    Person p = new Person();
}
```

During execution:

```text
GC Root
   |
   v
p
   |
   v
Person object
```

After `Test()` finishes, assume no other reference points to the object:

```text
GC Roots

(no path to Person object)

Person object
      X
   unreachable
```

The object is now **unreachable**.

It becomes:

> **eligible for garbage collection**

The GC can eventually reclaim the memory occupied by that object.

The important word is **eventually**.

The CLR does not necessarily reclaim the memory at the exact moment the object becomes unreachable.

---

# 8. Eligible for Collection Does Not Mean Collected Immediately

Suppose:

```csharp
Person p = new Person();
p = null;
```

Conceptually:

```text
Before:

p
 |
 v
Person object
```

After:

```text
p = null;

p
 |
 v
null

Person object
      |
      X
  unreachable
```

The object may now be eligible for garbage collection.

But this does **not** mean:

```text
p = null
   ↓
object immediately destroyed
```

Instead:

```text
p = null
   ↓
object may become unreachable
   ↓
eligible for GC
   ↓
GC eventually runs
   ↓
memory may be reclaimed
```

---

# 9. Multiple References Change the Lifetime

Consider:

```csharp
Person p1 = new Person();
Person p2 = p1;
```

Now there are two references to the same object:

```text
p1 --------+
           |
           v
       Person object
           ^
           |
p2 --------+
```

If we do:

```csharp
p1 = null;
```

the object is still reachable through `p2`:

```text
p1 → null

p2
 |
 v
Person object
```

Therefore the object is **not** eligible for collection.

Only when the last reachable reference is gone does the object become unreachable.

For example:

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

The object is now eligible for garbage collection.

---

# 10. Objects Can Keep Other Objects Alive

Consider:

```csharp
class Person
{
    public Address Address;
}
```

and:

```csharp
Person p = new Person();
p.Address = new Address();
```

Conceptually:

```text
GC Root
   |
   v
Person object
   |
   | Address reference
   v
Address object
```

The `Address` object is reachable because the `Person` object is reachable and the `Person` object contains a reference to it.

Now suppose:

```csharp
p = null;
```

and assume there are no other references:

```text
GC Roots

(no path)

Person object
     |
     v
Address object
```

Both objects are now unreachable.

This illustrates an important point:

> **Reachability can form an object graph.**

The GC is not simply looking at individual variables. It determines which objects can be reached through references.

---

# 11. The Object Graph

A group of objects connected through references can be viewed as an **object graph**.

For example:

```text
GC Root
   |
   v
Order
 |   \
 |    \
 v     v
Customer  Address
 |
 v
Account
```

The root provides a starting point.

The GC follows references:

```text
Root
 ↓
Order
 ↓
Customer
 ↓
Account
```

and:

```text
Order
 ↓
Address
```

All of these objects are reachable.

---

# 12. An Object Can Become Unreachable Even If Other Objects Still Exist

Consider:

```text
GC Root
   |
   v
Object A
   |
   v
Object B

Object C
```

Suppose `Object C` has no reference path from any GC root.

Then:

```text
Object A → Object B

Object C
   X
unreachable
```

The existence of `Object A` and `Object B` does not matter to `Object C`.

The GC cares about:

> **Can the object be reached from a GC root?**

---

# 13. Circular References Do Not Automatically Keep Objects Alive

Consider:

```text
Object A
   |
   v
Object B
   |
   +------> Object A
```

There is a circular reference.

A common misconception is:

> "Because A and B reference each other, they can never be collected."

That is false for a tracing garbage collector.

If there is no path from any GC root:

```text
GC Roots

(no path)

Object A ------> Object B
   ^                |
   |________________|
```

then both objects are unreachable.

Therefore both can eventually be collected.

The important question is not:

> "Do the objects reference each other?"

The important question is:

> **"Can the objects be reached from a GC root?"**

---

# 14. Why the GC Does Not Destroy Objects at Scope Exit

Consider:

```csharp
Person CreatePerson()
{
    Person p = new Person();
    return p;
}
```

If the runtime destroyed the object simply because the local variable went out of scope, returning an object would not work.

Instead:

```text
CreatePerson()

p
 |
 v
Person object
 |
 | return reference
 v
caller
 |
 v
same Person object
```

The object remains alive because it is still reachable.

Therefore:

> **A local variable going out of scope does not by itself determine object lifetime.**

---

# 15. `null` and Reachability

Setting a reference to `null` means that particular reference no longer points to the object.

Example:

```csharp
Person p = new Person();

p = null;
```

Before:

```text
p
 |
 v
Person object
```

After:

```text
p → null

Person object
      X
```

If there are no other references, the object becomes unreachable.

But if another reference exists:

```csharp
Person p1 = new Person();
Person p2 = p1;

p1 = null;
```

then:

```text
p1 → null

p2
 |
 v
Person object
```

The object remains reachable.

Therefore:

> **Setting one reference to `null` does not necessarily make an object eligible for collection.**

---

# 16. Garbage Collection Is About Memory Reclamation

The GC's fundamental job is to reclaim memory occupied by managed objects that are no longer reachable.

Conceptually:

```text
Object created
      ↓
Object reachable
      ↓
References disappear
      ↓
Object becomes unreachable
      ↓
Eligible for GC
      ↓
GC eventually reclaims memory
```

This is why managed memory does not normally require the programmer to explicitly free every object.

---

# 17. What the Programmer Should Not Assume

Do not assume:

```text
Method ends
    ↓
object destroyed
```

Do not assume:

```text
reference = null
    ↓
object immediately destroyed
```

Do not assume:

```text
object is unreachable
    ↓
GC immediately runs
```

Instead:

```text
Object becomes unreachable
        ↓
Object becomes eligible for GC
        ↓
GC decides when collection occurs
        ↓
Memory is eventually reclaimed
```

---

# 18. A Simple Example Bringing Everything Together

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

Now:

```csharp
void Test()
{
    Person p = new Person();
    p.Address = new Address();
}
```

Conceptually:

```text
GC Root
   |
   v
p
   |
   v
Person object
   |
   | Address reference
   v
Address object
```

When `Test()` finishes, assume no other references exist:

```text
GC Roots

(no path)

Person object
     |
     v
Address object
```

Both objects are unreachable.

Therefore:

```text
Person object
      ↓
eligible for GC

Address object
      ↓
eligible for GC
```

The GC can eventually reclaim the memory occupied by both objects.

---

# 19. The Key Questions to Ask

Whenever you see an object and want to understand its lifetime, ask:

### Question 1

> **Where is the reference to the object?**

### Question 2

> **Are there other references to the object?**

### Question 3

> **Can the object still be reached from a GC root?**

### Question 4

> **If no path exists, has the object merely become eligible for GC, or has GC actually reclaimed it?**

These questions are more useful than memorizing:

```text
"method ended = object destroyed"
```

because that statement is incorrect.

---

# 20. Final Mental Model

The complete object-lifetime model is:

```text
                 OBJECT CREATED
                       |
                       v
              Object is reachable
                       |
                       v
              References exist
                       |
             References disappear
                       |
                       v
              Object unreachable
                       |
                       v
             Eligible for GC
                       |
                       v
                GC eventually
                reclaims memory
```

The most important distinction is:

```text
Scope
  ↓
determines where a variable can be used

Reachability
  ↓
helps determine whether an object is alive

Garbage Collection
  ↓
reclaims memory for unreachable objects
```

## One sentence to remember

> **A managed object does not die simply because a variable goes out of scope; it becomes eligible for garbage collection when it is no longer reachable from any GC root, and the GC reclaims its memory when it performs collection.**
