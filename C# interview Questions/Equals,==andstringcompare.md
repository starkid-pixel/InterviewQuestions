# C# `==`, `Equals()`, and `string.Compare()`

## 1. Basic distinction

The most important distinction is:

| Syntax | What is it? | Purpose |
|---|---|---|
| `==` | Operator | Equality comparison |
| `Equals()` | Method | Equality comparison |
| `string.Compare()` | Method | Ordering/comparison |

Remember:

> **`==` is an operator. `Equals()` is a method.**

---

## 2. `==`

`==` is the equality operator.

### Value types

For value types, `==` generally compares values.

```csharp
int a = 10;
int b = 10;

Console.WriteLine(a == b); // true
```

### Reference types

For reference types, `==` compares references **by default**.

```csharp
Person p1 = new Person();
Person p2 = new Person();

Console.WriteLine(p1 == p2); // false
```

However, a reference type can **overload the `==` operator** and define its own equality semantics.

Therefore, the precise rule is:

> **For reference types, `==` compares references by default, but the type can overload `==`.**

---

## 3. `string` and `==`

`string` is a reference type, but it overloads the `==` operator.

Therefore:

```csharp
string a = "hello";
string b = "hello";

Console.WriteLine(a == b); // true
```

For `string`, `==` performs **content/value-based comparison**.

Conceptually:

```text
string
 ├── ==        → overloaded operator → compares contents
 └── Equals()  → overridden method    → compares contents
```

---

## 4. `Equals()`

`Equals()` is a **method**.

Its behavior depends on the type's implementation.

For an ordinary class, the default `Equals()` behavior is reference-based. A class can override `Equals()` to provide logical/value equality.

Example:

```csharp
class Person
{
    public int Id { get; set; }

    public override bool Equals(object? obj)
    {
        return obj is Person other && Id == other.Id;
    }
}
```

Therefore:

> **`Equals()` is a method whose implementation determines what equality means.**

When overriding `Equals()`, `GetHashCode()` must also be implemented consistently.

If:

```csharp
a.Equals(b) == true
```

then:

```csharp
a.GetHashCode() == b.GetHashCode()
```

must also be true.

This is especially important for:

```text
Dictionary<TKey, TValue>
HashSet<T>
```

---

## 5. `string.Equals()`

For strings, `Equals()` performs content-based equality.

```csharp
string a = "hello";
string b = "hello";

Console.WriteLine(a.Equals(b)); // true
```

You can also specify the comparison rules:

```csharp
string.Equals(
    a,
    b,
    StringComparison.OrdinalIgnoreCase);
```

This is useful when case sensitivity or culture rules matter.

---

## 6. `string.Compare()`

`string.Compare()` is a **method** primarily used for ordering.

```csharp
string a = "apple";
string b = "banana";

int result = string.Compare(a, b);
```

The result is interpreted as:

```text
< 0  → a comes before b
  0  → a and b are equal
> 0  → a comes after b
```

Therefore:

```text
== / Equals()
    → "Are these equal?"

string.Compare()
    → "Which one comes before/after?"
```

Example:

```csharp
if (string.Compare(a, b) < 0)
{
    // a comes before b
}
```

---

## 7. `StringComparison`

When string comparison semantics matter, explicitly specify `StringComparison`.

Common options include:

```csharp
StringComparison.Ordinal
StringComparison.OrdinalIgnoreCase
StringComparison.CurrentCulture
StringComparison.CurrentCultureIgnoreCase
```

For example:

```csharp
string.Equals(
    a,
    b,
    StringComparison.OrdinalIgnoreCase);
```

This is preferable to:

```csharp
a.ToLower() == b.ToLower()
```

because explicit comparison semantics make the intended behavior clear and avoid unnecessary string transformations.

---

## 8. Sonar Recommendation

Sonar may recommend `string.Compare()` or an explicit `StringComparison` depending on the exact code and rule being triggered.

This does **not** mean:

> "Always use `string.Compare()` instead of `==`."

Choose the API based on the intent.

### Simple equality

```csharp
a == b
```

or:

```csharp
string.Equals(a, b)
```

### Equality with explicit comparison rules

```csharp
string.Equals(
    a,
    b,
    StringComparison.OrdinalIgnoreCase);
```

### Ordering

```csharp
string.Compare(a, b)
```

---

## 9. Important `string` Summary

`string` is a **reference type**, but it provides value-based equality.

```text
string
 |
 +-- ==              → overloaded operator
 |                     → content equality
 |
 +-- Equals()        → overridden method
 |                     → content equality
 |
 +-- Compare()       → method
                       → ordering
```

So:

```csharp
string a = "hello";
string b = "hello";

a == b             // true
a.Equals(b)        // true
string.Compare(a,b) // 0
```

---

## 10. Interview Answer

A concise interview answer:

> **"`==` is an operator, while `Equals()` is a method. For value types, `==` generally compares values. For reference types, `==` compares references by default, unless the type overloads the operator. `string` is a reference type that overloads `==` and overrides `Equals()` to provide content-based equality. `string.Compare()` is a method used primarily when we need to determine ordering, returning a negative, zero, or positive value."**

### Quick Reference

| Operation | What is it? | Main purpose | `string` behavior |
|---|---|---|---|
| `a == b` | Operator | Equality | Content comparison |
| `a.Equals(b)` | Method | Equality | Content comparison |
| `string.Equals(a,b)` | Method | Equality | Content comparison |
| `string.Equals(a,b,StringComparison...)` | Method | Explicit equality rules | Content comparison with specified rules |
| `string.Compare(a,b)` | Method | Ordering | Returns `< 0`, `0`, or `> 0` |
| `ReferenceEquals(a,b)` | Method | Reference identity | Checks whether both refer to the same object |

## Key Points to Remember

1. **`==` is an operator.**
2. **`Equals()` is a method.**
3. Reference types use reference equality with `==` **by default**.
4. A type can **overload `==`**.
5. `string` overloads `==`.
6. `string` overrides `Equals()`.
7. Both `string ==` and `string.Equals()` compare **string contents**.
8. `string.Compare()` is mainly for **ordering**, not simply checking equality.
9. `Equals()` and `GetHashCode()` must be consistent when defining value equality.
