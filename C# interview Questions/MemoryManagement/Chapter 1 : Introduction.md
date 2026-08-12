# Introduction — C# Memory Allocation

## 1. Why Memory Allocation Becomes Confusing

C# memory allocation is often introduced with simple statements such as:

```text
Value type     → Stack
Reference type → Heap
```

These statements are useful as a first approximation, but they are **not the complete picture**.

They become confusing when we start looking at:

- value-type fields inside classes
- reference-type fields inside classes
- arrays
- arrays of value types
- arrays of reference types
- objects that contain references to other objects

The key is to stop thinking only in terms of **"stack versus heap"**.

Instead, ask two separate questions:

> **What is stored here?**

and

> **Where is that storage located?**

Once these two questions are separated, the memory model becomes much easier to understand.

---

# 2. Value Type vs Reference Type

The first distinction is what the variable or field actually contains.

## Value Type

A value type contains the **actual value**.

For example:

```csharp
int x = 10;
```

Conceptually:

```text
x
+------+
|  10  |
+------+
```

`x` contains the actual value `10`.

Examples of value types include:

- `int`
- `double`
- `bool`
- `struct`
- `enum`

---

## Reference Type

A reference-type variable contains a **reference to an object**.

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

`p` does not contain the complete `Person` object.

It contains a reference that identifies the `Person` object.

Examples of reference types include:

- `class`
- `string`
- arrays
- delegates

---

# 3. What Does "Inline" Mean?

The word **inline** is important when understanding value types.

When we say:

> A value is stored inline

we mean:

> **The actual value is stored directly as part of the storage occupied by its containing object, structure, or array.**

Consider:

```csharp
class Person
{
    public int Age;
}
```

If:

```csharp
Person p = new Person();
```

and:

```csharp
p.Age = 30;
```

then conceptually:

```text
Person object
+----------------+
| Age = 30       |
+----------------+
```

The actual `int` value `30` is part of the `Person` object.

There is no separate `int` object that the `Age` field refers to.

That is what **stored inline** means.

---

# 4. Reference-Type Fields Are Different

Now consider:

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

There are now two separate objects:

```text
Person object
+----------------------+
| Address → -----------|------> Address object
+----------------------+          +-------------+
                                  | Address data|
                                  +-------------+
```

The `Address` field does not contain the complete `Address` object.

It contains a **reference**.

The reference itself is part of the `Person` object.

The `Address` object is a separate object.

Therefore:

> **A reference-type field stores the reference inline inside its containing object; the referenced object is stored separately.**

---

# 5. One Managed Heap Does Not Mean One Object

The previous example sometimes creates another misunderstanding.

We may say:

> The `Person` object is on the managed heap and the `Address` object is also on the managed heap.

This does **not** mean there are two different heaps.

Conceptually, both objects can exist in the same managed heap:

```text
                    MANAGED HEAP

+------------------------------------------------+
|                                                |
| Person object          Address object          |
| +----------------+     +------------------+    |
| | Address -------|---->| Address data     |    |
| +----------------+     +------------------+    |
|                                                |
+------------------------------------------------+
```

They are:

- two separate objects
- independently allocated
- connected by a reference

So the correct statement is:

> **The `Person` object and the `Address` object are separate objects in managed memory. They are not necessarily located on separate heaps.**

---

# 6. When Is a Value Type Usually on the Stack?

Consider a value type used as a local variable:

```csharp
void Calculate()
{
    int x = 10;
    double price = 25.5;
    bool active = true;
}
```

For a simple conceptual model, we commonly represent these local variables as being in the method's stack frame:

```text
Method stack frame
+----------------+
| x = 10         |
| price = 25.5   |
| active = true  |
+----------------+
```

So it is reasonable to say:

> **A value-type local variable is commonly represented as being stored on the stack.**

However, this is a conceptual model.

The JIT compiler can use CPU registers or optimize a local variable away entirely.

Therefore, the more precise statement is:

> **Value-type local variables are commonly modeled as stack-resident, but their actual runtime storage is an implementation detail of the JIT.**

---

# 7. Is Every Value Type Stored on the Stack?

No.

This is one of the most important points.

Consider:

```csharp
class Person
{
    public int Age;
}
```

`Age` is still an `int`, and `int` is still a value type.

But `Age` is a **field inside the `Person` object**.

Conceptually:

```text
Managed Heap

Person object
+----------------+
| Age = 30       |
+----------------+
```

The value is stored inline inside the object.

It is not separately placed on the stack simply because `int` is a value type.

Therefore:

> **A value type does not automatically mean stack.**

Instead:

> **A value type stores its actual value inline in its containing storage.**

Where that containing storage exists depends on what contains the value.

---

# 8. The Same Value Type Can Be Stored in Different Places

Consider:

```csharp
int x = 10;
```

Here `x` is a local variable.

Conceptually:

```text
Stack
+---------+
| x = 10  |
+---------+
```

Now consider:

```csharp
class Person
{
    public int Age;
}
```

The `Age` field is part of a `Person` object:

```text
Managed Heap

Person object
+-------------+
| Age = 10    |
+-------------+
```

The type is still `int`.

What changed?

Not the value type.

What changed is **the containing storage**.

This gives us a much better mental model:

```text
VALUE TYPE
    |
    | contains actual value
    |
    +-----------------------------+
                                  |
                            containing storage
                                  |
                    +-------------+-------------+
                    |                           |
              local variable              object field
                    |                           |
              commonly stack              inside object
```

---

# 9. What Happens with a Struct?

A `struct` is also a value type.

```csharp
struct Point
{
    public int X;
    public int Y;
}
```

If it is a local:

```csharp
Point p = new Point();
```

conceptually:

```text
Local storage
+----------------+
| Point          |
| X = 10         |
| Y = 20         |
+----------------+
```

If the same struct is a field:

```csharp
class Shape
{
    public Point Position;
}
```

then:

```text
Shape object
+----------------+
| Position       |
|   X = 10       |
|   Y = 20       |
+----------------+
```

The `Point` value is stored inline inside the `Shape` object.

Again, the important idea is:

> **The value type is stored inline in whatever storage contains it.**

---

# 10. Class vs Struct — A Very Important Comparison

This distinction becomes especially clear if `Address` is changed from a class to a struct.

### `Address` as a class

```csharp
class Address
{
    public int Number;
}

class Person
{
    public Address Address;
}
```

Conceptually:

```text
Person object
+-----------------------+
| Address → ------------|----> Address object
+-----------------------+       +-------------+
                                | Number = 10 |
                                +-------------+
```

The `Person` object contains a **reference**.

The `Address` object is separate.

---

### `Address` as a struct

```csharp
struct Address
{
    public int Number;
}

class Person
{
    public Address Address;
}
```

Conceptually:

```text
Person object
+-----------------------+
| Address               |
|   Number = 10         |
+-----------------------+
```

The actual `Address` value is stored inline inside the `Person` object.

So:

```text
class Address
    ↓
reference type
    ↓
Person contains a reference
    ↓
separate Address object


struct Address
    ↓
value type
    ↓
Person contains the actual value
    ↓
stored inline
```

This is one of the best examples for understanding the difference between value types and reference types.

---

# 11. Arrays

Arrays are reference types.

Consider:

```csharp
Person[] people = new Person[3];
```

There are three things to understand:

1. `people` is a variable.
2. `new Person[3]` creates one array object.
3. Each element of the array is a reference to a `Person`.

Initially:

```text
people
   |
   | reference to array
   v
+----------------------+
| Person[] array       |
|                      |
| [0] = null           |
| [1] = null           |
| [2] = null           |
+----------------------+
```

At this point, **three `Person` objects have not been created**.

Only one array object has been created.

---

# 12. Creating the Person Objects

Now suppose:

```csharp
people[0] = new Person();
people[1] = new Person();
people[2] = new Person();
```

Now there is:

- one `Person[]` array object
- three separate `Person` objects

Conceptually:

```text
people
   |
   | reference to array
   v
+--------------------------------+
| Person[] array                 |
|                                |
| [0] reference -----------------|----> Person object #1
|                                |
| [1] reference -----------------|----> Person object #2
|                                |
| [2] reference -----------------|----> Person object #3
+--------------------------------+
```

The relationship is:

```text
people
   |
   v
Person[] array object
   |
   +---- reference → Person #1
   |
   +---- reference → Person #2
   |
   +---- reference → Person #3
```

This is **not** a two-heap situation.

It is one array object containing three references to three separate objects.

---

# 13. Why `Person[]` and `int[]` Are Different

Now compare:

```csharp
int[] numbers = new int[3];
```

with:

```csharp
Person[] people = new Person[3];
```

### `int[]`

The array elements are `int` values.

Conceptually:

```text
numbers
   |
   v
+----------------------+
| int[] array          |
|                      |
| [0] = 10             |
| [1] = 20             |
| [2] = 30             |
+----------------------+
```

The actual values are stored inline in the array.

### `Person[]`

The array elements are references.

```text
people
   |
   v
+----------------------+
| Person[] array       |
|                      |
| [0] → Person         |
| [1] → Person         |
| [2] → Person         |
+----------------------+
```

The array contains references to `Person` objects.

Therefore:

> **An array stores its elements inline. What each element contains depends on the array's element type.**

Examples:

```text
int[]       → actual int values
Point[]     → actual Point values
Person[]    → references to Person objects
string[]    → references to string objects
```

This is another excellent example of why **value type does not simply mean stack**.

---

# 14. The Important Difference Between a Variable and an Object

Consider:

```csharp
Person p = new Person();
```

There are two separate things:

```text
p
```

and:

```text
new Person()
```

`p` is the variable.

`new Person()` creates the object.

Conceptually:

```text
Local variable
+---------+
| p -------|-----------------> Person object
+---------+
```

Similarly:

```csharp
Person[] people = new Person[3];
```

contains:

```text
Local variable
+----------------+
| people --------|-----------------> Person[] array object
+----------------+
```

Therefore, never confuse:

> **the variable**

with:

> **the object referred to by the variable.**

This distinction is essential for understanding memory allocation.

---

# 15. A Better Mental Model

Instead of memorizing:

```text
Value type     → Stack
Reference type → Heap
```

use this model:

```text
                    WHAT IS STORED?
                           |
              +------------+------------+
              |                         |
         VALUE TYPE               REFERENCE TYPE
              |                         |
        actual value               reference
              |                         |
              v                         v
         stored inline             points to object
                                      |
                                      v
                               separate object
```

Then ask:

```text
WHERE IS THE CONTAINING STORAGE?
```

For example:

```text
Local value
    ↓
commonly stack/register

Value-type field
    ↓
inside containing object/structure

Reference-type field
    ↓
reference stored inside containing object/structure

Array element
    ↓
stored inside the array object
```

---

# 16. The Core Rules

| Question | Correct mental model |
|---|---|
| What does a value type contain? | The actual value |
| What does a reference type variable contain? | A reference to an object |
| What does "inline" mean? | The value/reference is stored directly as part of its containing storage |
| When is a value type commonly on the stack? | When it is a local variable, subject to JIT implementation |
| Where is a value-type field? | The actual value is stored inline inside its containing object/structure |
| Where is a reference-type field? | The reference is stored inline inside its containing object/structure |
| Where is the referenced object? | Separately, generally in managed memory |
| Where is an array? | The array itself is an object and is generally in managed memory |
| What does `Person[]` contain? | References to `Person` objects |
| What does `int[]` contain? | Actual `int` values |
| Does `Person[] people = new Person[3]` create three `Person` objects? | No. It creates one array containing three initially-null references |
| Does a class field containing another class mean two heaps? | No. It means two separate objects that can exist in the same managed heap |
| Does value type always mean stack? | No |
| Does reference type mean the variable itself is on the heap? | No |

---

# 17. The One Mental Model to Remember

The entire introduction can be reduced to this:

```text
                    TYPE
                     |
          +----------+----------+
          |                     |
      VALUE TYPE            REFERENCE TYPE
          |                     |
      actual value           reference
          |                     |
          v                     v
      stored inline        stored inline
      in its container     in its container
                                |
                                v
                           separate object
```

And then:

```text
WHERE IS THE CONTAINER?

Local variable
    ↓
commonly stack/register

Object field
    ↓
inside the object

Array element
    ↓
inside the array
```

Therefore:

> **Do not start memory allocation with "value type = stack" and "reference type = heap." Start with what is stored and what contains it.**

Once that distinction is clear, stack, heap, objects, fields, arrays, structs, references, and garbage collection become much easier to understand.
