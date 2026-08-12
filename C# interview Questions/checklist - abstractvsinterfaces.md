# C# Abstract Class vs Interface — Decision Checklist

## Abstract Class — Checklist

Consider an **abstract class** when you answer **Yes** to one or more of these:

- [ ] Do the classes have a **strong common relationship**?
- [ ] Do they need to share **common state/data**?
- [ ] Do they need to share **common implementation**?
- [ ] Do I want to provide some **default/common behavior**?
- [ ] Do I want to define some behavior that derived classes **must specialize**?
- [ ] Do I need a **base implementation that derived classes can extend or override**?
- [ ] Does the **Template Method** pattern fit the problem?
- [ ] Would duplicating the common implementation across classes be undesirable?

### Think

> **"These classes belong to the same family and should share implementation/state."**

---

## Interface — Checklist

Consider an **interface** when you answer **Yes** to one or more of these:

- [ ] Do I primarily need to define a **contract**?
- [ ] Am I describing a **capability**?
- [ ] Could **unrelated classes** need this capability?
- [ ] Do I want **multiple classes to provide their own implementations**?
- [ ] Does the class already need to inherit from another base class?
- [ ] Do I need a class to support **multiple contracts/capabilities**?
- [ ] Do I want **loose coupling** between the consumer and implementation?
- [ ] Do I want easy **Dependency Injection / mocking / testing**?
- [ ] Could there be **multiple interchangeable implementations**?

### Think

> **"I don't care what the class is; I care that it can perform this capability/contract."**

---

## Quick Decision Tree

```text
Do I need shared state or substantial shared implementation?
                |
          +-----+-----+
         YES          NO
          |            |
          v            v
    Abstract Class   Do I need a
                    contract/capability?
                         |
                    +----+----+
                   YES        NO
                    |          |
                    v          v
                Interface   Reconsider
                            the abstraction
```

---

## The Best Interview Shortcut

Remember these **4 questions**:

| Question | Yes → |
|---|---|
| Do they share **state**? | Abstract Class |
| Do they share **implementation**? | Abstract Class |
| Do they share a **contract/capability**? | Interface |
| Can **unrelated classes** implement it? | Interface |

---

## The Core Distinction

> **Abstract class: "What common implementation can I provide?"**

> **Interface: "What contract/capability do I require?"**

---

## Important C# Nuance

These are not absolute rules.

Modern C# interfaces can contain **default implementations**, so the decision should not be based only on whether an interface can technically contain implementation.

The real architectural question is:

> **What relationship and responsibility am I modeling?**

Use an **abstract class** when the abstraction represents a related family of classes that should share state and/or implementation.

Use an **interface** when the abstraction represents a contract or capability that different classes can implement independently.
