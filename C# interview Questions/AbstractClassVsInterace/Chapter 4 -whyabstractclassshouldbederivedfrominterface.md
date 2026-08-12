# Interface → Abstract Class → Concrete Class

## Why Do We Sometimes Derive an Abstract Class from an Interface?

A common C# design pattern is:

```text
Interface
    ↓
Abstract Class
    ↓
Concrete Class
```

For example:

```csharp
public interface IRepository
{
    void Add();
    void Delete();
    void Update();
}
```

Then:

```csharp
public abstract class RepositoryBase : IRepository
{
    public virtual void Add()
    {
        // Common implementation
    }

    public virtual void Delete()
    {
        // Common implementation
    }

    public virtual void Update()
    {
        // Common implementation
    }
}
```

And finally:

```csharp
public class CustomerRepository : RepositoryBase
{
    public override void Add()
    {
        // Customer-specific implementation
    }
}
```

There are **three different concepts** working together here.

---

# 1. Interface Defines the Contract

The interface says:

> **"Any repository must provide Add, Delete and Update."**

```text
IRepository
     |
     +-- Add()
     +-- Delete()
     +-- Update()
```

The interface establishes **what must be supported**, not the shared implementation.

---

# 2. Abstract Class Provides the Common Implementation

The abstract class says:

> **"I will provide a default/common implementation so derived classes don't have to duplicate the code."**

```text
             IRepository
                  |
                  v
            RepositoryBase
                  |
       +----------+----------+
       |          |          |
     Add()     Delete()    Update()
       |
    common
 implementation
```

This is where the abstract class becomes useful.

The abstract class acts as a **reusable implementation foundation** for classes that implement the interface.

---

# 3. `virtual` Gives Derived Classes a Choice

When you write:

```csharp
public virtual void Add()
{
    // Common implementation
}
```

you are saying:

> **"Here is the default implementation, but a derived class is allowed to replace it."**

For example:

```csharp
public class CustomerRepository : RepositoryBase
{
    public override void Add()
    {
        // Customer-specific behavior
    }
}
```

The derived class can override the behavior.

But it doesn't have to:

```csharp
public class ProductRepository : RepositoryBase
{
    // Uses RepositoryBase.Add()
}
```

Therefore:

> **`virtual` = default implementation + optional customization**

---

# 4. Why Not Make the Methods Abstract?

You could instead write:

```csharp
public abstract class RepositoryBase : IRepository
{
    public abstract void Add();
    public abstract void Delete();
    public abstract void Update();
}
```

But now every derived class **must** implement all three methods.

```text
RepositoryBase
       |
       +-- Add()       MUST implement
       +-- Delete()    MUST implement
       +-- Update()    MUST implement
```

You lose the benefit of shared implementation.

With `virtual`:

```text
RepositoryBase
       |
       +-- Add()       default implementation
       |                  |
       |                  +--> override if needed
       |
       +-- Delete()    default implementation
       |
       +-- Update()    default implementation
```

This gives you more flexibility.

---

# 5. Why Have Both an Interface and an Abstract Class?

This is the deeper architectural reason.

They serve **different consumers**.

## Interface — Useful to Consumers

A consumer can depend only on the contract:

```csharp
IRepository repository;
```

The consumer doesn't need to know how the repository is implemented.

```text
Consumer
   |
   v
IRepository
```

The actual implementation could be:

```text
RepositoryBase
CustomerRepository
MockRepository
RemoteRepository
```

The consumer only depends on the abstraction.

---

## Abstract Class — Useful to Implementers

The abstract class provides a reusable foundation:

```text
                 IRepository
                      |
                      v
                RepositoryBase
                      |
          +-----------+-----------+
          |                       |
 CustomerRepository        ProductRepository
```

The derived classes can reuse the common implementation and customize only what they need.

So a useful mental model is:

> **Interface = contract for consumers**

> **Abstract class = reusable foundation for implementers**

---

# 6. Real-World Example

Consider a notification system.

First, define the contract:

```csharp
public interface INotificationSender
{
    void Send(string message);
}
```

Then provide a reusable base implementation:

```csharp
public abstract class NotificationSenderBase : INotificationSender
{
    public virtual void Send(string message)
    {
        Validate(message);
        Log(message);
        SendInternal(message);
    }

    protected void Validate(string message)
    {
        // Common validation
    }

    protected void Log(string message)
    {
        // Common logging
    }

    protected abstract void SendInternal(string message);
}
```

Then create concrete implementations:

```csharp
public class EmailNotificationSender : NotificationSenderBase
{
    protected override void SendInternal(string message)
    {
        // Email-specific implementation
    }
}
```

And:

```csharp
public class SmsNotificationSender : NotificationSenderBase
{
    protected override void SendInternal(string message)
    {
        // SMS-specific implementation
    }
}
```

The design becomes:

```text
                 INotificationSender
                         |
                    CONTRACT
                         |
                         v
              NotificationSenderBase
                         |
                COMMON IMPLEMENTATION
                         |
              +----------+----------+
              |                     |
              v                     v
           Email                    SMS
        specialized             specialized
        implementation           implementation
```

This allows the common behavior to live in one place while specialized behavior remains in the concrete classes.

---

# 7. `abstract` vs `virtual`

This distinction is important.

## `abstract`

> **"You MUST provide the implementation."**

```csharp
public abstract void Process();
```

There is no implementation in the abstract base member.

The derived class must implement it:

```csharp
public override void Process()
{
    // Required implementation
}
```

---

## `virtual`

> **"I provide a default implementation, but you MAY replace it."**

```csharp
public virtual void Process()
{
    // Default implementation
}
```

A derived class can override it:

```csharp
public override void Process()
{
    // Customized implementation
}
```

But overriding is optional.

---

## Normal Method

A normal non-virtual method provides an implementation that derived classes cannot override polymorphically.

```csharp
public void Process()
{
    // Fixed implementation
}
```

---

# 8. The Complete Pattern

When you see:

```text
Interface
    ↓
Abstract Class
    ↓
Concrete Class
```

and the abstract class contains `virtual` methods, think:

```text
Interface
    ↓
Defines WHAT must be supported
    ↓
Abstract Class
    ↓
Provides common/default HOW
    ↓
Virtual
    ↓
Allows derived classes to customize
    ↓
Concrete Class
    ↓
Provides specific behavior where necessary
```

This is a deliberate combination of:

- **Abstraction**
- **Contract**
- **Code reuse**
- **Polymorphism**
- **Extensibility**

---

# 9. Why This Is a Powerful Design

The pattern gives you three useful levels of responsibility:

| Level | Responsibility |
|---|---|
| Interface | Defines the **contract** |
| Abstract class | Provides **shared/default implementation** |
| Concrete class | Provides **specific implementation/customization** |

For example:

```text
IRepository
    ↓
What must a repository do?

RepositoryBase
    ↓
What common behavior can all repositories reuse?

CustomerRepository
    ↓
What behavior is specific to customers?
```

This avoids duplicating common code while still allowing specialized behavior.

---

# 10. When Should You Use This Pattern?

Consider this pattern when:

- [ ] You need a public contract for consumers.
- [ ] Multiple implementations share substantial behavior.
- [ ] You want to avoid duplicating common implementation.
- [ ] Some behavior should be common to all implementations.
- [ ] Some behavior needs to be specialized by derived classes.
- [ ] You want derived classes to optionally override default behavior.
- [ ] You want consumers to depend on an interface rather than a concrete implementation.

A useful mental model is:

> **Interface → What must be supported**

> **Abstract class → What can be shared**

> **Virtual member → What can be customized**

> **Concrete class → What is specific to this implementation**

---

# 11. Interview Answer

If an interviewer asks:

> **"Why would you derive an abstract class from an interface and then make its methods virtual?"**

A strong answer is:

> **"The interface defines the contract that consumers depend on. The abstract class provides a reusable base implementation of that contract for related implementations. Making the members virtual provides default behavior while allowing derived classes to override it when they need specialized behavior. This gives us a combination of contract, code reuse, and extensibility."**

---

# 12. Final Mental Model

```text
                    INTERFACE
                        ↓
                 Defines the contract
                        ↓
                 ABSTRACT CLASS
                        ↓
              Provides shared behavior
                        ↓
                    virtual
                        ↓
            Allows optional customization
                        ↓
                 CONCRETE CLASS
                        ↓
             Provides specific behavior
```

## One-Line Summary

> **Interface defines the contract, abstract class provides reusable implementation, and virtual members allow derived classes to customize that implementation.**
