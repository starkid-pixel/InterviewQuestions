# 14. Abstraction

**Abstraction** means hiding implementation details and exposing only the essential functionality.

For example, a payment system may expose:

```
payment.Process();
```

The caller doesn't need to know whether the payment is processed using:

-  Credit card
-  UPI
-  Bank transfer
-  Wallet
-  Other payment systems

```
             Payment
                |
        +-------+-------+
        |               |
   Credit Card         UPI
        |               |
   Implementation   Implementation
```

The user of the payment service only needs to know the contract.

---

# 15. C++ and Interfaces

C++ does **not** have a dedicated `interface` keyword like C#.

C++ typically achieves interface-like behavior using:

-  Abstract classes
-  Pure virtual functions

Example:

```
class IAnimal
{
public:
    virtual void MakeSound() = 0;

    virtual ~IAnimal() = default;
};
```

The following is a **pure virtual function**:

```
virtual void MakeSound() = 0;
```

The `= 0` means that derived classes are required to provide an implementation.

---

# 16. C++ Abstract Class

Because `IAnimal` contains a pure virtual function, it is an abstract class.

You cannot instantiate it:

```
// ❌ Not allowed
IAnimal animal;
```

But you can create a derived class:

```
class Dog : public IAnimal
{
public:
    void MakeSound() override
    {
        cout << "Bark";
    }
};
```

Then:

```
Dog dog;

dog.MakeSound();
```

---

# 17. C++ Interface-Like Class

A C++ interface-like class usually contains mostly pure virtual functions.

Example:

```
class ILogger
{
public:
    virtual void Log(string message) = 0;

    virtual ~ILogger() = default;
};
```

Implementation:

```
class FileLogger : public ILogger
{
public:
    void Log(string message) override
    {
        // Write to file
    }
};
```

Conceptually:

```
C# Interface
      |
      v
C++ Abstract Class
      |
      v
Pure Virtual Functions
      |
      v
Interface-like behavior
```

---

# 18. C++ Abstract Class vs Interface-Like Class

C++ doesn't enforce a separate interface type.

The distinction is primarily about design.

## Interface-Like Class

Usually contains:

-  Pure virtual functions
-  Little or no state
-  Virtual destructor
-  A contract for derived classes

Example:

```
class ILogger
{
public:
    virtual void Log(string message) = 0;

    virtual ~ILogger() = default;
};
```

It means:

> "Any class implementing this contract must provide logging behavior."

---

## Abstract Base Class

Can contain:

-  Pure virtual functions
-  Concrete methods
-  Member variables
-  Constructors
-  Destructors
-  Protected members

Example:

```
class Animal
{
protected:
    string name;

public:

    Animal(string name)
        : name(name)
    {
    }

    void Eat()
    {
        cout << name << " is eating";
    }

    virtual void MakeSound() = 0;

    virtual ~Animal() = default;
};
```

This provides:

```
Common State
     +
Common Implementation
     +
Required Behavior
```

---

# 19. C++ Multiple Inheritance

C++ supports multiple inheritance.

Therefore, a class can implement multiple interface-like abstract classes.

```
class IFlyable
{
public:
    virtual void Fly() = 0;

    virtual ~IFlyable() = default;
};

class ISwimmable
{
public:
    virtual void Swim() = 0;

    virtual ~ISwimmable() = default;
};
```

A class can inherit from both:

```
class Duck : public IFlyable, public ISwimmable
{
public:

    void Fly() override
    {
        cout << "Flying";
    }

    void Swim() override
    {
        cout << "Swimming";
    }
};
```

Conceptually:

```
       IFlyable       ISwimmable
           \             /
            \           /
                Duck
```

---

# 20. C++ IS-A vs CAN-DO

A useful design rule:

## Abstract Base Class → IS-A

```
Dog IS-A Animal
Car IS-A Vehicle
Circle IS-A Shape
```

Example:

```
class Dog : public Animal
{
};
```

## Interface-Like Class → CAN-DO

```
Bird CAN Fly
Duck CAN Swim
Printer CAN Print
Logger CAN Log
```

Example:

```
class Bird : public IFlyable
{
};
```

A class can combine both:

```
class Duck : public Animal, public IFlyable, public ISwimmable
{
};
```

Meaning:

```
Duck
 |
 +-- IS-A Animal
 |
 +-- CAN Fly
 |
 +-- CAN Swim
```

---

# 21. C# Abstract Class

An abstract class in C# is declared using the `abstract` keyword.

```
abstract class Animal
{
    public string Name { get; set; }

    public void Eat()
    {
        Console.WriteLine("Eating...");
    }

    public abstract void MakeSound();
}
```

It cannot be instantiated directly:

```
// ❌ Not allowed
Animal animal = new Animal();
```

A derived class must implement the abstract method:

```
class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Bark");
    }
}
```

Usage:

```
Dog dog = new Dog();

dog.Eat();
dog.MakeSound();
```

---

# 22. What Can an Abstract Class Contain?

An abstract class can contain:

-  Abstract methods
-  Concrete methods
-  Fields
-  Properties
-  Constructors
-  Static members
- `private` members
- `protected` members
- `public` members
-  Common state
-  Common behavior

Example:

```
abstract class Employee
{
    protected decimal salary;

    public Employee(decimal salary)
    {
        this.salary = salary;
    }

    public void CalculateBonus()
    {
        Console.WriteLine(salary * 0.10m);
    }

    public abstract void Work();
}
```

---

# 23. C# Interface

An interface defines a **contract**.

```
interface IFlyable
{
    void Fly();
}
```

A class implements it:

```
class Bird : IFlyable
{
    public void Fly()
    {
        Console.WriteLine("Flying...");
    }
}
```

The interface says:

> If a class implements `IFlyable`, it must provide the required `Fly()` behavior.

---

# 24. Multiple Interfaces in C#

A class can implement multiple interfaces.

```
interface IFlyable
{
    void Fly();
}

interface ISwimmable
{
    void Swim();
}
```

A class can implement both:

```
class Duck : IFlyable, ISwimmable
{
    public void Fly()
    {
        Console.WriteLine("Flying");
    }

    public void Swim()
    {
        Console.WriteLine("Swimming");
    }
}
```

Conceptually:

```
       IFlyable       ISwimmable
           \             /
            \           /
                Duck
```

---

# 25. Abstract Class vs Interface in C#

| Feature | Abstract Class | Interface |
|---|---|---|
| Purpose | Shared base class | Contract/capability |
| Class inheritance | One base class only | Multiple interfaces allowed |
| Instance fields | Yes | No |
| Instance state | Yes | No |
| Constructor | Yes | No instance constructor |
| Abstract members | Yes | Yes, interface members can require implementation |
| Concrete methods | Yes | Yes, modern C# supports default implementations |
| Properties | Yes | Yes |
| Protected members | Yes | No |
| Shared implementation | Natural fit | Possible with default interface implementations |
| Multiple contracts | Limited by single class inheritance | Yes |

---

# 26. Modern C# Interfaces

Older explanations often say:

> "An interface can only contain method declarations."

This is no longer completely correct.

Modern C# interfaces can provide **default implementations**.

Example:

```
interface ILogger
{
    void Log(string message);

    void LogError(string message)
    {
        Log("ERROR: " + message);
    }
}
```

Therefore, a better definition is:

> **An interface defines a contract that types can implement.**

Do not define an interface simply as:

> "A collection of methods with no implementation."

---

# 27. When to Use Abstract Class

Use an abstract class when:

## 1. Classes are closely related

```
Animal
 ├── Dog
 ├── Cat
 └── Horse
```

## 2. They share state

```
abstract class Employee
{
    protected string name;
    protected decimal salary;
}
```

## 3. They share implementation

```
abstract class Employee
{
    public void CalculateSalary()
    {
        // Common implementation
    }

    public abstract void Work();
}
```

## 4. You need constructors

```
abstract class Animal
{
    protected string name;

    protected Animal(string name)
    {
        this.name = name;
    }
}
```

## 5. You need protected members

```
abstract class Vehicle
{
    protected int speed;
}
```

---

# 28. When to Use Interface

Use an interface when you want to define a **capability or contract**.

Example:

```
interface IPrintable
{
    void Print();
}
```

Different and unrelated classes can implement it:

```
class Invoice : IPrintable
{
    public void Print()
    {
    }
}

class Photo : IPrintable
{
    public void Print()
    {
    }
}

class Report : IPrintable
{
    public void Print()
    {
    }
}
```

Conceptually:

```
Invoice ──┐
          |
Photo ────┼──> IPrintable
          |
Report ───┘
```

The classes don't need to have the same base class.

---

# 29. IS-A vs CAN-DO

This is one of the easiest ways to remember the difference.

## Abstract Class

Ask:

> **What is this object?**

Examples:

```
Dog IS-A Animal
Car IS-A Vehicle
Circle IS-A Shape
Manager IS-A Employee
```

---

## Interface

Ask:

> **What can this object do?**

Examples:

```
Bird CAN Fly
Duck CAN Swim
Printer CAN Print
PaymentProcessor CAN ProcessPayment
Logger CAN Log
```

---

# 30. Simple Mental Model

## Abstract Class

```
              Vehicle
                 |
        +--------+--------+
        |                 |
       Car              Bike

Vehicle provides:
- Common state
- Start()
- Stop()
- Common behavior
- Required behavior
```

## Interface

```
        IPrintable
        /    |    \
       /     |     \
   Invoice  Photo  Report

IPrintable says:

"These objects can be printed."
```

---

# 31. C++ vs C#

| Concept | C++ | C# |
|---|---|---|
| Dedicated interface keyword | No | Yes |
| Abstract class | Yes | Yes |
| Pure virtual function | `virtual ... = 0` | No equivalent syntax |
| Interface-like abstraction | Abstract class + pure virtual functions | `interface` |
| Multiple class inheritance | Yes | No |
| Multiple interfaces | Multiple inheritance | Yes |
| Member variables in abstract class | Yes | Yes |
| Concrete methods | Yes | Yes |
| Constructors in abstract class | Yes | Yes |
| Virtual destructor | Common for polymorphic base classes | Managed automatically |

---

# 32. C++ vs C# Interface Example

## C++

```
class IAnimal
{
public:
    virtual void MakeSound() = 0;

    virtual ~IAnimal() = default;
};
```

## C#

```
interface IAnimal
{
    void MakeSound();
}
```

Conceptually:

```
C++
 |
 +-- Abstract Class
       |
       +-- Pure Virtual Function
               |
               v
       Interface-like behavior

C#
 |
 +-- interface
       |
       +-- Contract
```

---

# 33. Interview Questions

## Q1. What is CLR?

### Answer

> CLR stands for Common Language Runtime. It is the execution engine of the .NET platform. It runs managed .NET code and provides services such as JIT compilation, garbage collection, memory management, exception handling, type safety, threading, synchronization, and runtime security mechanisms.

---

## Q2. What does JIT do?

### Answer

> JIT stands for Just-In-Time compiler. It converts Intermediate Language (IL) into native machine code that can be executed by the CPU.

```
C#
 ↓
IL
 ↓
JIT
 ↓
Machine Code
 ↓
CPU
```

---

## Q3. What is Garbage Collection?

### Answer

> Garbage Collection automatically identifies unreachable managed objects and reclaims their memory.

---

## Q4. Does CLR provide security?

### Answer

> Yes. The CLR and .NET platform provide runtime security mechanisms such as type safety and managed execution. However, application-level security such as authentication, authorization, encryption, and input validation is still the developer's responsibility.

---

## Q5. Does CLR support threading?

### Answer

> Yes. The .NET runtime and libraries provide support for threads, thread pools, tasks, synchronization primitives, and asynchronous programming.

---

## Q6. Does C++ have interfaces?

### Answer

> C++ does not have a dedicated `interface` keyword. Interface-like behavior is commonly implemented using abstract classes with pure virtual functions.

---

## Q7. What is a pure virtual function in C++?

### Answer

```
virtual void MakeSound() = 0;
```

A function declared with `= 0` is a pure virtual function.

A class containing a pure virtual function is abstract and cannot normally be instantiated.

---

## Q8. What is the difference between an abstract class and an interface in C#?

### Answer

> An abstract class is used when related classes need to share common state and implementation, while an interface defines a contract or capability that can be implemented by different classes. A class can inherit from only one base class but can implement multiple interfaces.

---

# 34. Final Cheat Sheet

## CLR

```
                         .NET
                           |
                           v
                         CLR
                           |
        +------------------+------------------+
        |                  |                  |
       JIT                 GC             Threading
        |                  |                  |
    IL -> Machine       Memory            Threads/Tasks
        |
        +-- Exception Handling
        +-- Type Safety
        +-- Runtime Security
        +-- Interoperability
        +-- Synchronization
```

## Abstraction

```
                 ABSTRACTION
                     |
          +----------+----------+
          |                     |
          v                     v
   Abstract Class          Interface
          |                     |
   Shared state &          Contract /
   implementation          capability
          |                     |
       "IS-A"                "CAN-DO"
          |                     |
      Dog IS-A              Dog CAN Fly
      Animal                Dog CAN Swim
```

---

# 35. Key Takeaways

### CLR

> **CLR is the runtime engine of .NET.**

It provides:

```
JIT
GC
Memory Management
Exception Handling
Threading
Synchronization
Type Safety
Runtime Security
Interoperability
```

### Abstraction

> **Abstraction means hiding implementation details and exposing essential behavior.**

### C++ Interface

> **C++ has no dedicated interface keyword. Interface-like behavior is usually implemented using abstract classes and pure virtual functions.**

### C# Abstract Class

> **An abstract class is useful when related classes share state, behavior, and a common base identity.**

### C# Interface

> **An interface defines a contract or capability that multiple unrelated or related classes can implement.**

### Easy Rule

```
Abstract Class
       |
       v
"What ARE you?"
       |
       v
Dog IS-A Animal

Interface
       |
       v
"What CAN you do?"
       |
       v
Dog CAN Fly
Dog CAN Swim
Dog CAN Run
```

## One-Sentence Summary

> **CLR is the .NET runtime that executes and manages managed code; abstraction hides implementation details; an abstract class provides shared identity, state, and implementation; an interface defines a contract or capability; and C++ achieves interface-like behavior through abstract classes and pure virtual functions.**
