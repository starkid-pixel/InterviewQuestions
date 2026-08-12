# C# Abstract Class vs Interface — Detailed Decision Guide

## 1. Start with the Fundamental Question

When you encounter a design problem, don't start by asking:

> "Should I use an abstract class or interface?"

Start with:

> **"What relationship am I trying to model?"**

There are two fundamentally different situations.

### Situation A — Shared Implementation

You have a group of related classes that share:

- state
- behavior
- implementation
- lifecycle
- common rules

Then an **abstract class** is a natural candidate.

```text
                 Employee
                    |
          +---------+---------+
          |                   |
      Developer             Manager
```

For example:

```csharp
public abstract class Employee
{
    public string Name { get; set; }

    public void Login()
    {
        // Common implementation
    }

    public abstract void PerformWork();
}
```

`Developer` and `Manager` are both employees, and therefore it makes sense for them to inherit common employee behavior.

The abstract class can say:

> "Here is the stuff that every employee gets automatically, and here is the stuff you must specialize."

---

## 2. Interface Is a Different Thought Process

Now imagine:

```text
            IPrintable
           /    |     \
          /     |      \
      Report  Invoice  Image
```

These objects don't necessarily have a common base class.

But they all have one capability:

> **They can be printed.**

So:

```csharp
public interface IPrintable
{
    void Print();
}
```

Now:

```csharp
public class Report : IPrintable
{
    public void Print()
    {
        // Report-specific printing
    }
}

public class Image : IPrintable
{
    public void Print()
    {
        // Image-specific printing
    }
}
```

The interface doesn't care **how** printing happens.

It says:

> "If you claim to be `IPrintable`, you must provide `Print()`."

That's the essence of an interface.

---

# 3. The Key Distinction: Implementation vs Contract

This is the most important thing to understand.

### Abstract Class

The base class can say:

> "I know how some of this works. I'll give you the implementation."

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

The base class owns part of the implementation.

```text
             DataProcessor
                  |
        +---------+---------+
        |                   |
    CsvProcessor       XmlProcessor

Shared:
    Process()

Specialized:
    Load()
    Validate()
    Save()
```

This is a strong indication for an abstract class.

---

### Interface

An interface normally says:

> "I don't provide the object's identity or shared state. I define what it must be capable of doing."

```csharp
public interface IDataExporter
{
    void Export(Data data);
}
```

You could have:

```text
IDataExporter
     |
     +---- CsvExporter
     |
     +---- JsonExporter
     |
     +---- XmlExporter
     |
     +---- PdfExporter
```

Each implementation can be completely different internally.

---

# 4. Why Shared State Is Important

This is one of the strongest reasons to choose an abstract class.

Suppose:

```csharp
public abstract class Vehicle
{
    protected int Speed;
    protected string RegistrationNumber;

    public void Start()
    {
        // Common implementation
    }

    public abstract void Drive();
}
```

Every vehicle has:

- `Speed`
- `RegistrationNumber`
- `Start()`

So the base class can hold those things.

An interface cannot provide that kind of shared instance state.

Therefore:

> **If common state is an important part of the abstraction, think abstract class.**

---

# 5. Why Multiple Interfaces Are Important

Consider:

```text
Developer
   |
   +--- IEmployee
   +--- IPrintable
   +--- IAuditable
   +--- IReportable
```

C# allows:

```csharp
public class Developer : Employee,
                         IPrintable,
                         IAuditable,
                         IReportable
{
}
```

This is extremely useful in application architecture.

The class has **one primary inheritance hierarchy**, but can participate in **multiple contracts**.

That's why interfaces work particularly well for:

- services
- repositories
- handlers
- plugins
- adapters
- dependency injection
- infrastructure boundaries

---

# 6. A Very Important Question: "Could These Classes Be Unrelated?"

This is a very good interface test.

Suppose:

```csharp
public interface ILoggable
{
    void Log();
}
```

You could have:

```text
Customer      → ILoggable
Order         → ILoggable
Payment       → ILoggable
FileProcessor → ILoggable
```

These objects don't have to belong to the same family.

The commonality is simply:

> They support logging.

That's a strong interface scenario.

---

# 7. Another Important Question: "Do I Need Inheritance?"

Suppose:

```csharp
public abstract class Shape
{
    public int X { get; set; }
    public int Y { get; set; }

    public abstract void Draw();
}
```

Then:

```text
Circle : Shape
Rectangle : Shape
Triangle : Shape
```

There is a meaningful inheritance relationship.

They share:

- coordinates
- possibly common rendering behavior
- shape-related rules

An abstract class makes sense.

But:

```csharp
IPrintable
```

doesn't imply a common class hierarchy.

It simply means those things **can be printed**.

---

# 8. Dependency Injection Is Another Strong Interface Scenario

This is very common in .NET.

Suppose:

```csharp
public interface IPaymentService
{
    void ProcessPayment();
}
```

You can have:

```csharp
public class CreditCardPaymentService : IPaymentService
{
    public void ProcessPayment()
    {
    }
}

public class PayPalPaymentService : IPaymentService
{
    public void ProcessPayment()
    {
    }
}
```

Your consumer doesn't need to know which implementation it receives:

```csharp
public class OrderService
{
    private readonly IPaymentService _paymentService;

    public OrderService(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
}
```

This creates:

```text
OrderService
      |
      | depends on
      v
IPaymentService
      ^
      |
 +----+----------------+
 |                     |
CreditCard          PayPal
```

This is **programming against an abstraction**.

The interface creates a boundary between the consumer and implementation.

---

# 9. Don't Make the Mistake of Thinking Interface = No Implementation

This is important for modern C#.

Interfaces can have default implementations:

```csharp
public interface ILogger
{
    void Log(string message);

    void LogError(string message)
    {
        Log("ERROR: " + message);
    }
}
```

So technically:

> **"Interface cannot have implementation"**

is no longer completely correct.

The better architectural distinction is:

> **Abstract class is primarily useful for shared state and reusable implementation.**

> **Interface is primarily useful for defining a contract/capability and enabling independent implementations.**

---

# 10. Detailed Decision Checklist

When designing something, mentally run through these questions.

### Question 1 — Do These Classes Share Meaningful State?

```text
YES → Consider Abstract Class
NO  → Continue
```

### Question 2 — Do They Share Substantial Implementation?

```text
YES → Consider Abstract Class
NO  → Continue
```

### Question 3 — Are They Conceptually a Family of Related Classes?

```text
YES → Consider Abstract Class
NO  → Continue
```

### Question 4 — Am I Defining a Contract or Capability?

```text
YES → Consider Interface
NO  → Continue
```

### Question 5 — Could Unrelated Classes Implement This Capability?

```text
YES → Interface
```

### Question 6 — Could a Class Need Several Such Capabilities?

```text
YES → Interface
```

### Question 7 — Do I Want Interchangeable Implementations?

```text
YES → Interface is usually a strong candidate
```

---

# 11. Decision in One Picture

```text
                     Start
                       |
                       v
             What am I modeling?
                       |
             +---------+---------+
             |                   |
             v                   v
       Common family?       Capability/Contract?
             |                   |
            YES                 YES
             |                   |
             v                   v
       Shared state?          Interface
             |
            YES
             |
             v
       Abstract Class
```

Do not treat this as a rigid algorithm.

A common family may also have interfaces:

```csharp
public abstract class Employee
{
    // Shared state + implementation
}

public interface IReportable
{
    void GenerateReport();
}

public class Developer : Employee, IReportable
{
}
```

This is often the **best design**:

```text
             Employee
                |
            Developer
                |
          +-----+-----+
          |           |
    IReportable    IAuditable
```

The abstract class handles the **common implementation**.

The interfaces handle **additional contracts/capabilities**.

---

# 12. Interview Answer

If an interviewer asks:

> **"When would you choose an abstract class over an interface?"**

A strong senior-level answer would be:

> **"I would choose an abstract class when the types have a strong common relationship and need to share state or implementation. I would choose an interface when I want to define a contract or capability that can be implemented independently by different types, including unrelated types. An abstract class gives me a shared implementation base, whereas interfaces give me composable contracts and allow multiple capabilities."**

This demonstrates **design understanding**, rather than simply remembering C# syntax.

---

# 13. Final Mental Model

```text
Abstract Class
        ↓
Shared state
        +
Shared implementation
        +
Related family of classes


Interface
        ↓
Contract
        +
Capability
        +
Independent implementations
        +
Multiple capabilities
```

## Final Rule

> **Choose an abstract class when you want to share state and implementation across a related family of classes.**

> **Choose an interface when you want to define a contract or capability that can be implemented independently by different classes.**
