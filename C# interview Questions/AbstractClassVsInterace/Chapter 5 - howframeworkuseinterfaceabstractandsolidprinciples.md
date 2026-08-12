# SOLID Principles + Template Method Pattern

## 1. Why Connect SOLID and Template Method?

SOLID principles and design patterns should not be studied as completely separate topics.

SOLID gives us **design principles** that help us decide how responsibilities, dependencies, and extension points should be structured.

Design patterns provide **reusable structures** for solving recurring design problems.

The **Template Method Pattern** is a good example because it demonstrates how several SOLID principles can work together.

The pattern commonly looks like:

```text
Abstract Base Class
        |
        +-- Defines the overall algorithm
        |
        +-- Provides common implementation
        |
        +-- Leaves selected steps customizable
                    |
                    v
              Concrete Classes
```

---

# 2. Template Method Pattern

The Template Method Pattern defines the **overall structure of an algorithm in a base class** while allowing derived classes to customize selected steps.

Example:

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

The algorithm is fixed:

```text
Process()
   |
   +-- Load()
   |
   +-- Validate()
   |
   +-- Save()
```

The subclasses decide how each step is implemented.

```csharp
public class CsvProcessor : DataProcessor
{
    protected override void Load()
    {
        // Load CSV
    }

    protected override void Validate()
    {
        // Validate CSV
    }

    protected override void Save()
    {
        // Save CSV
    }
}
```

Another implementation:

```csharp
public class XmlProcessor : DataProcessor
{
    protected override void Load()
    {
        // Load XML
    }

    protected override void Validate()
    {
        // Validate XML
    }

    protected override void Save()
    {
        // Save XML
    }
}
```

The base class controls **what happens and in what order**.

The derived classes control **how the individual steps happen**.

---

# 3. The Core Idea of Template Method

The most important mental model is:

> **Base class owns the algorithm; derived classes provide the variable parts.**

```text
                DataProcessor
                     |
             Process() algorithm
                     |
          +----------+----------+
          |          |           |
        Load()    Validate()    Save()
          |          |           |
          +----------+-----------+
                     |
              specialized by
              derived classes
```

This is useful when the algorithm has:

- a common sequence
- common rules
- common infrastructure
- predictable steps
- a few variation points

---

# 4. `abstract` vs `virtual` in Template Method

Template Method can use both `abstract` and `virtual` members.

## Abstract Member

Use `abstract` when the derived class **must provide the behavior**.

```csharp
protected abstract void Save();
```

Meaning:

> **"Every derived class must define Save()."**

---

## Virtual Member

Use `virtual` when the base class provides a **default implementation**, but derived classes may customize it.

```csharp
protected virtual void Log()
{
    Console.WriteLine("Processing started");
}
```

Meaning:

> **"This is the default behavior; override it only if necessary."**

This distinction gives the pattern flexibility.

```text
abstract → mandatory variation point

virtual  → optional variation point
```

---

# 5. Interface + Abstract Class + Template Method

A very common enterprise design can combine all three:

```text
Interface
    ↓
Abstract Base Class
    ↓
Concrete Implementations
```

For example:

```csharp
public interface IDataProcessor
{
    void Process();
}
```

Then:

```csharp
public abstract class DataProcessorBase : IDataProcessor
{
    public void Process()
    {
        Load();
        Validate();
        Log();
        Save();
    }

    protected abstract void Load();

    protected abstract void Validate();

    protected virtual void Log()
    {
        Console.WriteLine("Processing data");
    }

    protected abstract void Save();
}
```

Concrete implementation:

```csharp
public class CsvProcessor : DataProcessorBase
{
    protected override void Load()
    {
        // Load CSV
    }

    protected override void Validate()
    {
        // Validate CSV
    }

    protected override void Save()
    {
        // Save CSV
    }
}
```

Now the responsibilities are separated:

```text
IDataProcessor
      ↓
Defines the contract
      ↓
DataProcessorBase
      ↓
Defines the algorithm + common behavior
      ↓
CsvProcessor
      ↓
Provides CSV-specific behavior
```

---

# 6. SOLID Connection

Template Method does not automatically make a design SOLID.

However, it can be designed in a way that supports several SOLID principles.

---

# 7. Single Responsibility Principle (SRP)

SRP says:

> **A class should have one reason to change.**

In Template Method, the base class can own the **common workflow**, while subclasses own the **specific variation**.

For example:

```text
DataProcessorBase
    → controls processing workflow

CsvProcessor
    → handles CSV-specific behavior

XmlProcessor
    → handles XML-specific behavior
```

The responsibilities are separated.

If the common workflow changes, the base class changes.

If CSV parsing changes, `CsvProcessor` changes.

If XML parsing changes, `XmlProcessor` changes.

That is a useful application of SRP.

### Important Nuance

Don't interpret SRP as:

> "One class must have only one method."

SRP is about **responsibility and reason for change**, not method count.

---

# 8. Open/Closed Principle (OCP)

This is one of the strongest connections.

OCP says:

> **Software entities should be open for extension but closed for modification.**

Suppose the system initially supports:

```text
CSV
XML
```

Later you need:

```text
JSON
```

You can create:

```csharp
public class JsonProcessor : DataProcessorBase
{
    protected override void Load()
    {
        // Load JSON
    }

    protected override void Validate()
    {
        // Validate JSON
    }

    protected override void Save()
    {
        // Save JSON
    }
}
```

The existing processing algorithm does not need to change.

You extended the system by adding a new implementation.

```text
Existing:
DataProcessorBase
CsvProcessor
XmlProcessor

Extension:
JsonProcessor
```

This is a strong example of OCP.

### The Important Insight

The abstract base class creates **variation points**.

```text
Fixed:
Process()
  ↓
Load()
  ↓
Validate()
  ↓
Save()

Variable:
Load implementation
Validate implementation
Save implementation
```

The fixed parts remain stable.

The variable parts can be extended.

---

# 9. Liskov Substitution Principle (LSP)

LSP says:

> **Objects of a derived type should be usable wherever the base type is expected without breaking the expected behavior.**

If:

```csharp
DataProcessor processor = new CsvProcessor();
processor.Process();
```

then `CsvProcessor` must honor the expectations established by `DataProcessor`.

A derived class should not:

- break the workflow
- violate important base-class assumptions
- produce behavior that contradicts the base contract
- throw unexpected exceptions for valid operations unless that is part of the contract

The Template Method pattern helps establish a controlled extension model.

The base class controls the algorithm.

Derived classes customize designated steps.

```text
Base class
    ↓
Controls invariants
    ↓
Derived class
    ↓
Provides variation
```

This can make LSP easier to maintain.

### Important Nuance

Template Method does **not guarantee LSP**.

A poorly designed derived class can still violate LSP.

For example, if the base class expects `Save()` to always persist valid data, a derived class that silently does nothing may violate the behavioral contract.

---

# 10. Interface Segregation Principle (ISP)

ISP says:

> **Clients should not be forced to depend on interfaces they do not use.**

Template Method itself does not directly solve ISP.

But when the interface is introduced, we should keep it focused.

Good:

```csharp
public interface IDataProcessor
{
    void Process();
}
```

Less desirable:

```csharp
public interface IDataProcessor
{
    void Process();
    void Export();
    void Print();
    void SendEmail();
    void GenerateReport();
}
```

If different processors don't need all those operations, the interface becomes too broad.

Instead, use focused interfaces where appropriate:

```csharp
public interface IDataProcessor
{
    void Process();
}

public interface IExportable
{
    void Export();
}
```

The key lesson:

> **Template Method doesn't automatically provide ISP; the surrounding abstractions still need to be designed carefully.**

---

# 11. Dependency Inversion Principle (DIP)

DIP says:

> **High-level modules should not depend directly on low-level modules; both should depend on abstractions.**

Suppose:

```csharp
public interface IDataProcessor
{
    void Process();
}
```

A high-level service can depend on it:

```csharp
public class ProcessingService
{
    private readonly IDataProcessor _processor;

    public ProcessingService(IDataProcessor processor)
    {
        _processor = processor;
    }

    public void Execute()
    {
        _processor.Process();
    }
}
```

The service doesn't depend directly on:

```text
CsvProcessor
XmlProcessor
JsonProcessor
```

It depends on:

```text
IDataProcessor
```

The implementations can then be supplied through Dependency Injection.

```text
             ProcessingService
                    |
                    v
             IDataProcessor
                    ^
          +---------+---------+
          |         |         |
        CSV        XML       JSON
```

This is where Template Method, interfaces, DI, and DIP can work together.

---

# 12. Template Method and Polymorphism

Template Method relies heavily on polymorphism.

The base class calls methods whose implementations are supplied by derived classes.

```csharp
public void Process()
{
    Load();
    Validate();
    Save();
}
```

At runtime:

```text
Process()
   |
   +--> CsvProcessor.Load()
   |
   +--> CsvProcessor.Validate()
   |
   +--> CsvProcessor.Save()
```

or:

```text
Process()
   |
   +--> XmlProcessor.Load()
   |
   +--> XmlProcessor.Validate()
   |
   +--> XmlProcessor.Save()
```

The caller invokes the same `Process()` method.

The behavior varies through polymorphism.

---

# 13. Template Method and Strategy Pattern

These two patterns are often confused.

## Template Method

Uses **inheritance**.

```text
Base class
    |
    +-- Derived class
```

The algorithm structure is controlled by the base class.

```text
Process()
  ↓
Step 1
  ↓
Step 2
  ↓
Step 3
```

Derived classes customize steps.

---

## Strategy

Uses **composition**.

```text
Context
   |
   +---- Strategy
             |
       +-----+-----+
       |           |
   Strategy A   Strategy B
```

The behavior itself is supplied as an object.

For example:

```csharp
public interface ICompressionStrategy
{
    byte[] Compress(byte[] data);
}
```

The context can receive a strategy:

```csharp
public class FileProcessor
{
    private readonly ICompressionStrategy _strategy;

    public FileProcessor(ICompressionStrategy strategy)
    {
        _strategy = strategy;
    }
}
```

### Core difference

> **Template Method → vary parts of an algorithm through inheritance.**

> **Strategy → replace an algorithm/behavior through composition.**

This distinction becomes important in architecture.

---

# 14. Template Method vs Simple Inheritance

Not every abstract base class is Template Method.

This:

```csharp
public abstract class Employee
{
    public abstract void Work();
}
```

is simply an abstract base class.

It does not necessarily implement Template Method.

Template Method normally has:

1. A method defining the algorithm.
2. A sequence of steps.
3. Some fixed steps.
4. Some variation points.
5. Derived classes implementing/customizing those variation points.

Example:

```csharp
public void Process()
{
    Load();
    Validate();
    Save();
}
```

That is the Template Method.

---

# 15. A More Realistic Enterprise Example

Consider document processing.

Requirements:

- Load document
- Validate document
- Transform document
- Save document
- Log processing

Different formats have different implementations.

```csharp
public abstract class DocumentProcessor
{
    public void Process()
    {
        LogStart();

        Load();
        Validate();
        Transform();
        Save();

        LogEnd();
    }

    protected virtual void LogStart()
    {
        Console.WriteLine("Processing started");
    }

    protected abstract void Load();

    protected abstract void Validate();

    protected abstract void Transform();

    protected abstract void Save();

    protected virtual void LogEnd()
    {
        Console.WriteLine("Processing completed");
    }
}
```

Now:

```text
                    DocumentProcessor
                           |
                     Process()
                           |
       +-------------------+-------------------+
       |                   |                   |
     Load()             Validate()          Transform()
       |                   |                   |
       +-------------------+-------------------+
                           |
                         Save()
```

A PDF processor:

```csharp
public class PdfProcessor : DocumentProcessor
{
    protected override void Load()
    {
        // Load PDF
    }

    protected override void Validate()
    {
        // Validate PDF
    }

    protected override void Transform()
    {
        // Transform PDF
    }

    protected override void Save()
    {
        // Save PDF
    }
}
```

A Word processor:

```csharp
public class WordProcessor : DocumentProcessor
{
    protected override void Load()
    {
        // Load Word document
    }

    protected override void Validate()
    {
        // Validate Word document
    }

    protected override void Transform()
    {
        // Transform Word document
    }

    protected override void Save()
    {
        // Save Word document
    }
}
```

The common workflow remains centralized.

---

# 16. Where `virtual` Fits

Not every variation point needs to be mandatory.

Suppose logging is optional:

```csharp
protected virtual void LogStart()
{
    Console.WriteLine("Processing started");
}
```

A derived class can simply inherit it.

If a specific implementation needs different logging:

```csharp
protected override void LogStart()
{
    // Custom logging
}
```

So:

```text
abstract
    ↓
Mandatory variation

virtual
    ↓
Optional variation
```

This is a very useful design distinction.

---

# 17. Common Mistakes

## Mistake 1 — Making Everything Virtual

Don't make every method virtual just because derived classes might someday need customization.

Every virtual member creates an extension point.

Too many extension points can make the design difficult to reason about.

Use `virtual` intentionally.

---

## Mistake 2 — Making Everything Abstract

If the base class has genuinely common behavior, don't force every derived class to duplicate it.

Prefer:

```csharp
protected virtual void Log()
{
    // Common behavior
}
```

when customization is optional.

Use:

```csharp
protected abstract void Save();
```

when customization is mandatory.

---

## Mistake 3 — Putting Too Much Business Logic in the Base Class

A base class can become a "god class" if it accumulates too much responsibility.

For example:

```text
DocumentProcessorBase
    |
    +-- database logic
    +-- network logic
    +-- authentication
    +-- logging
    +-- validation
    +-- file handling
    +-- business rules
    +-- UI logic
```

That defeats the purpose of good separation.

Template Method should control the workflow, not become a dumping ground for unrelated responsibilities.

---

# 18. SOLID + Template Method Summary

| Principle | How Template Method Can Support It |
|---|---|
| **SRP** | Base class owns common workflow; subclasses own specialized steps |
| **OCP** | Add new derived implementations without changing the common algorithm |
| **LSP** | Derived classes can substitute the base type if they honor its behavioral contract |
| **ISP** | Focused interfaces can keep contracts small; Template Method itself does not guarantee ISP |
| **DIP** | An interface can allow higher-level code to depend on abstractions rather than concrete implementations |

The important point is:

> **A design pattern does not automatically make code SOLID.**

The pattern provides a structure.

The developer still has to apply the principles correctly.

---

# 19. The Bigger Architecture Picture

This is where the concepts start connecting:

```text
                    SOLID
                      |
          +-----------+-----------+
          |           |           |
         SRP         OCP         DIP
          |           |           |
          +-----------+-----------+
                      |
                Design Pattern
                      |
              Template Method
                      |
        +-------------+-------------+
        |             |             |
   Abstract Base   Polymorphism  Variation Points
        |
        +-------------+
        |
   Concrete Classes
```

And when Dependency Injection is added:

```text
                    DIP
                     |
                     v
                 Interface
                     |
                     v
              Dependency Injection
                     |
                     v
                    IoC
```

So these concepts are not isolated topics.

They form a chain of design thinking.

---

# 20. Senior-Level Mental Model

Instead of memorizing the pattern as:

> "Template Method uses an abstract class."

Think:

> **"I have a stable algorithm with controlled variation points. I want to centralize the stable workflow and allow implementations to customize only the parts that vary."**

Then ask:

1. What is stable?
2. What varies?
3. Who should own the stable workflow?
4. Which steps must be implemented?
5. Which steps have reasonable defaults?
6. Should the variation be implemented through inheritance or composition?
7. Does the resulting design respect SOLID principles?

This is the level at which Template Method becomes an **architecture/design tool**, rather than just a design-pattern definition.

---

# 21. Final Interview Answer

If asked:

> **"Explain Template Method and its relationship with SOLID."**

A strong answer is:

> **"Template Method defines the skeleton of an algorithm in a base class and allows derived classes to customize selected steps. The base class owns the stable workflow, while subclasses implement the variation points. This can support SRP by separating common workflow from specialized behavior, OCP by allowing new implementations without modifying the existing algorithm, and LSP when derived classes correctly honor the base contract. If an interface is introduced, DIP can also be applied by allowing higher-level consumers to depend on the abstraction rather than concrete implementations. The pattern itself does not guarantee SOLID; it is a structure that can be used to implement SOLID principles well."**

---

# 22. Final Mental Model

```text
SOLID
  ↓
Design Principles
  ↓
Identify what should remain stable
  +
Identify what should vary
  ↓
Template Method
  ↓
Abstract Class
  ↓
Stable Algorithm
  +
Controlled Variation Points
  ↓
Concrete Implementations
```

## One-Line Summary

> **Template Method uses an abstract base class to keep the algorithm stable while allowing controlled variation in derived classes; when designed carefully, it can support several SOLID principles, especially SRP, OCP, and LSP.**
