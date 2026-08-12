# C# Abstract Class vs Interface

## Short Definitions

### Abstract Class

> An abstract class provides **shared state and implementation** for a family of related classes, while allowing them to provide specialized behavior.

### Interface

> An interface is a **contract or capability** that defines what a class must provide, allowing different classes to have their own implementations.

## One-Line Mental Model

> **Abstract class → shared state + shared implementation**

> **Interface → contract + capability + independent implementation**

## Key Difference

| Aspect | Abstract Class | Interface |
|---|---|---|
| Core idea | Shared state and implementation | Contract / capability |
| Represents | Common base with reusable behavior | What a class can do |
| State | Can contain instance state | Cannot contain instance state |
| Shared behavior | Can provide substantial shared implementation | Primarily defines a contract; modern C# also supports default implementations |
| Inheritance | A class can inherit only one base/abstract class | A class can implement multiple interfaces |
| Best suited for | Closely related classes that share state and implementation | Classes that need a common contract or capability, even when their implementations differ |

## Abstract Class

An abstract class is useful when several classes are related and need to share common state or implementation.

```csharp
public abstract class Employee
{
    public string Name { get; set; }

    public void DisplayName()
    {
        Console.WriteLine(Name);
    }

    public abstract void CalculateSalary();
}
```

Here, `Employee` provides:

- **Shared state** → `Name`
- **Shared implementation** → `DisplayName()`
- **Specialized behavior** → `CalculateSalary()`

Derived classes reuse the common implementation and provide their own specialized behavior.

```text
Employee
   |
   +---- Developer
   |
   +---- Manager
```

## Interface

An interface is primarily used to define a contract or capability.

```csharp
public interface IPrintable
{
    void Print();
}
```

Different classes can implement the interface independently:

```csharp
public class Report : IPrintable
{
    public void Print()
    {
        Console.WriteLine("Printing report");
    }
}

public class Invoice : IPrintable
{
    public void Print()
    {
        Console.WriteLine("Printing invoice");
    }
}
```

The important point is that `Report` and `Invoice` can have completely different implementations while satisfying the same contract.

```text
          IPrintable
          /        \
         /          \
      Report       Invoice
```

## The Most Important Difference

Think of the distinction this way:

### Abstract Class

> **Shared state + shared implementation**

Use an abstract class when related classes need to reuse common state and behavior.

### Interface

> **Contract + capability + independent implementation**

Use an interface when different classes need to provide the same capability or contract, without requiring them to share a common implementation.

## Why Multiple Interfaces Matter

C# supports single class inheritance:

```csharp
public class Developer : Employee
{
}
```

A class cannot inherit from multiple classes:

```csharp
// Not allowed
public class Developer : Employee, Manager
{
}
```

However, a class can implement multiple interfaces:

```csharp
public class Developer : Employee, IPayable, IPrintable, ILoggable
{
}
```

This allows an abstract/base class to provide the common implementation while interfaces add additional capabilities.

```text
                    Employee
                       |
                    Developer
                   /    |     \
            IPayable IPrintable ILoggable
```

## When to Use an Abstract Class

Use an abstract class when:

- Classes are closely related.
- They share common state.
- They share substantial implementation.
- There is a meaningful common base abstraction.
- You want to provide reusable behavior while forcing derived classes to implement specific behavior.

### Example: Template Method Pattern

```csharp
public abstract class DataProcessor
{
    public void Process()
    {
        Load();
        Validate();
        Save();
    }

    protected abstract void Load();
    protected abstract void Validate();
    protected abstract void Save();
}
```

The abstract class controls the overall algorithm and provides the common structure, while subclasses provide the specialized implementation.

## When to Use an Interface

Use an interface when:

- You need to define a contract.
- You want to represent a capability.
- Different or unrelated classes may implement the contract.
- You want multiple types of capabilities on a class.
- You want loose coupling and interchangeable implementations.
- You want to support Dependency Injection and easier testing.

### Example

```csharp
public interface ILogger
{
    void Log(string message);
}
```

Different implementations can satisfy the same contract:

```csharp
public class FileLogger : ILogger
{
    public void Log(string message)
    {
        // Write to file
    }
}

public class DatabaseLogger : ILogger
{
    public void Log(string message)
    {
        // Write to database
    }
}
```

The interface defines **what must be provided**, while each implementation decides **how it is provided**.

## Interview-Level Example

Consider a payment system:

```csharp
public abstract class Payment
{
    public decimal Amount { get; set; }

    public void ValidateAmount()
    {
        // Common validation
    }

    public abstract void Process();
}
```

And an additional capability:

```csharp
public interface IRefundable
{
    void Refund();
}
```

A concrete payment type can combine both:

```csharp
public class CreditCardPayment : Payment, IRefundable
{
    public override void Process()
    {
        // Credit card processing
    }

    public void Refund()
    {
        // Credit card refund
    }
}
```

Here:

- `Payment` provides **shared state and implementation**.
- `IRefundable` provides a **capability/contract**.
- `CreditCardPayment` provides the **specialized implementation**.

```text
Payment
   |
   +--- CreditCardPayment
             |
             +--- IRefundable
```

## Important Interview Nuance

Do not choose an interface simply because:

> "Interfaces are better than abstract classes."

The real design question is:

> **Do I need shared state and reusable implementation, or do I need to define a contract/capability that different classes can implement independently?**

That is the important distinction.

## Final Interview Answer

> **An abstract class provides shared state and implementation for a family of related classes, while an interface defines a contract or capability that different classes can implement independently.**
