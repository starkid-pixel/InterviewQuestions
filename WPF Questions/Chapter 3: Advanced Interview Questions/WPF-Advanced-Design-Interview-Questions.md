# WPF Advanced Design Interview Questions

> **LEVEL: ADVANCED**
>
> **Target Audience:** Senior WPF Developer | Lead Developer | Software Architect
>
> This chapter focuses on architecture, internals, performance, concurrency, scalability, maintainability, and real-world design scenarios.

---

## 1. Advanced WPF Architecture

### Q1. How would you architect a WPF application with 200+ screens?

A strong architecture separates presentation, application orchestration, domain/business logic, and infrastructure.

```text
WPF Views / Controls
        |
        v
ViewModels / Presentation
        |
        v
Application Services
        |
        +------> Domain
        |
        +------> Infrastructure
                   |
                   +--> REST/API
                   +--> Database
                   +--> File System
                   +--> Messaging
```

For a very large application, introduce modules:

```text
Shell
 |
 +-- Module A
 +-- Module B
 +-- Module C
 +-- Shared UI
 +-- Shared Infrastructure
```

Avoid having every ViewModel depend directly on every other module.

---

### Q2. How do you prevent architectural erosion?

Use explicit boundaries and enforce them through:

- project/module dependencies
- interfaces
- code reviews
- architecture tests
- dependency rules
- shared coding standards
- automated CI checks

For example:

```text
Presentation -> Application -> Domain
Infrastructure -> Application/Domain
Domain -> Nothing UI-specific
```

A ViewModel should not directly contain SQL queries or HTTP implementation details.

---

### Q3. How would you migrate a legacy WPF application to MVVM?

Do not rewrite everything at once.

A safer approach:

1. Add tests around critical behavior.
2. Identify business logic hidden in code-behind.
3. Extract services.
4. Introduce ViewModels for new screens.
5. Gradually move logic from existing code-behind.
6. Introduce dependency injection.
7. Replace tightly coupled components incrementally.

This is a **strangler/refactoring approach**, not a big-bang rewrite.

---

## 2. Advanced MVVM

### Q4. How do you prevent ViewModels from becoming God objects?

Split responsibilities.

Instead of:

```csharp
class CustomerViewModel
{
    // API
    // validation
    // database
    // navigation
    // dialogs
    // reporting
    // caching
    // business rules
}
```

use focused services:

```csharp
public interface ICustomerService { }
public interface INavigationService { }
public interface IDialogService { }
public interface IValidationService { }
public interface IReportService { }
```

The ViewModel coordinates these services instead of implementing everything.

---

### Q5. Should a ViewModel know about a View?

Preferably no.

Avoid:

```csharp
var window = new CustomerWindow();
window.Show();
```

Instead:

```csharp
await navigationService.NavigateAsync<CustomerViewModel>();
```

The infrastructure maps the ViewModel to the appropriate View.

This improves testing and reduces coupling.

---

### Q6. How should unrelated ViewModels communicate?

Possible approaches:

- shared application service
- mediator
- event aggregator
- message bus

Use an event aggregator carefully. Excessive event-based communication can make control flow difficult to understand and can introduce memory leaks if subscriptions are not removed.

---

## 3. Advanced Data Binding

### Q7. Why is a WPF binding not working?

Check:

1. `DataContext`
2. binding path
3. property visibility
4. `INotifyPropertyChanged`
5. collection notifications
6. converter
7. relative source
8. element name
9. dependency property
10. Output window binding errors

Example:

```xml
<TextBox Text="{Binding Customer.Name, UpdateSourceTrigger=PropertyChanged}" />
```

If `Customer.Name` changes dynamically, the appropriate object must raise `PropertyChanged`.

---

### Q8. Why doesn't ObservableCollection update when an item's property changes?

`ObservableCollection<T>` observes collection operations:

```text
Add
Remove
Move
Replace
Reset
```

It does not automatically observe properties of its items.

Therefore:

```csharp
ObservableCollection<Customer>
```

does not detect:

```csharp
customer.Name = "New Name";
```

The `Customer` itself should implement `INotifyPropertyChanged`.

---

## 4. Dependency Properties

### Q9. Why does WPF use DependencyProperty?

Dependency properties provide WPF features such as:

- binding
- styles
- animation
- inheritance
- default values
- property metadata
- change callbacks
- coercion

Example:

```csharp
public static readonly DependencyProperty TitleProperty =
    DependencyProperty.Register(
        nameof(Title),
        typeof(string),
        typeof(MyControl),
        new PropertyMetadata(string.Empty));

public string Title
{
    get => (string)GetValue(TitleProperty);
    set => SetValue(TitleProperty, value);
}
```

---

### Q10. What is property value precedence?

A simplified view is:

```text
Animation
    ↓
Local value
    ↓
TemplatedParent / Template
    ↓
Implicit/explicit styles and triggers
    ↓
Theme styles
    ↓
Inherited/default values
```

The exact precedence rules matter when diagnosing why a property does not have the value you expect.

---

### Q11. When would you use coercion?

Use coercion when a value must remain within a valid range.

Example:

```csharp
private static object CoerceValue(
    DependencyObject d,
    object baseValue)
{
    var value = (double)baseValue;
    return Math.Max(0, Math.Min(100, value));
}
```

This is useful for properties such as percentage, opacity-like values, or bounded numeric settings.

---

## 5. Commands

### Q12. How do you implement an async command safely?

A common pattern is:

```csharp
public async Task SaveAsync()
{
    if (IsBusy)
        return;

    try
    {
        IsBusy = true;
        await service.SaveAsync();
    }
    finally
    {
        IsBusy = false;
    }
}
```

A reusable `AsyncCommand` can encapsulate this behavior.

Important concerns:

- duplicate execution
- cancellation
- exceptions
- progress
- `CanExecute`
- UI state

---

### Q13. How do you prevent a user from clicking Save five times?

Use command state or an execution gate.

```csharp
if (IsBusy)
    return;

try
{
    IsBusy = true;
    await SaveAsync();
}
finally
{
    IsBusy = false;
}
```

For highly concurrent workflows, use an appropriate synchronization primitive such as `SemaphoreSlim`.

---

## 6. WPF Threading and Dispatcher

### Q14. Why can't a background thread update a WPF control?

WPF UI objects have thread affinity. Most UI objects must be accessed from the Dispatcher/UI thread.

Instead of:

```csharp
Task.Run(() =>
{
    textBox.Text = "Done"; // unsafe
});
```

marshal the update:

```csharp
await Dispatcher.InvokeAsync(() =>
{
    textBox.Text = "Done";
});
```

Prefer keeping UI access at the presentation boundary rather than passing controls into background services.

---

### Q15. Does async/await create a new thread?

No.

`async` and `await` do not inherently create threads.

For I/O:

```csharp
await httpClient.GetAsync(url);
```

the operation can wait without blocking the UI thread.

`Task.Run` is appropriate for CPU-bound work when moving that work away from the UI thread is beneficial.

---

### Q16. What is an async deadlock?

A classic problem is:

```csharp
var result = GetDataAsync().Result;
```

A UI thread may block waiting for an asynchronous operation whose continuation is trying to return to that same UI synchronization context.

Prefer:

```csharp
var result = await GetDataAsync();
```

Use async all the way through the call chain.

---

## 7. High-Performance WPF

### Q17. How do you optimize a DataGrid containing millions of records?

Do not load millions of objects into the UI.

Use:

```text
Database
   ↓
Filtering
   ↓
Sorting
   ↓
Paging / virtualization
   ↓
API/service
   ↓
WPF DataGrid
```

Important techniques:

- server-side filtering
- server-side sorting
- pagination
- UI virtualization
- data virtualization
- incremental loading
- debounced search
- cancellation
- lightweight DataTemplates

---

### Q18. How do you investigate a WPF UI freeze?

Do not guess.

Use a measurement-driven approach:

```text
Reproduce
   ↓
Capture CPU / UI trace
   ↓
Identify blocking operation
   ↓
Check synchronous I/O
   ↓
Check long CPU work
   ↓
Check layout/rendering
   ↓
Check excessive collection updates
   ↓
Fix
   ↓
Measure again
```

Potential causes include:

- `.Result` / `.Wait()`
- database calls on UI thread
- HTTP calls on UI thread
- huge collection updates
- expensive converters
- expensive DataTemplates
- excessive layout passes
- synchronous loops

---

## 8. Memory Leaks

### Q19. How can events cause WPF memory leaks?

If a long-lived publisher references a short-lived subscriber:

```csharp
publisher.Changed += subscriber.OnChanged;
```

the publisher can keep the subscriber alive.

For long-lived publishers, consider:

- unsubscribe
- weak events
- scoped lifetime
- explicit disposal

---

### Q20. How can DispatcherTimer cause a memory leak?

A timer can keep its target alive through its event handler.

For example:

```csharp
timer.Tick += OnTick;
```

If the timer lives longer than the ViewModel, the ViewModel may remain reachable.

Stop and detach long-lived timers during cleanup.

---

### Q21. How do you investigate a memory leak?

Use a memory profiler.

Look for:

- objects that should have died but remain alive
- GC roots
- event subscriptions
- static references
- timers
- caches
- global services
- retained visual trees

The key question is:

> What is keeping this object reachable?

---

## 9. Styles, Templates and Resources

### Q22. StaticResource vs DynamicResource?

`StaticResource` resolves the resource when the relevant XAML is loaded.

`DynamicResource` maintains a runtime resource reference and is useful when resources can change dynamically, such as runtime theme switching.

Use `DynamicResource` where runtime resource replacement is required; otherwise `StaticResource` is generally simpler.

---

### Q23. DataTemplate vs ControlTemplate?

`DataTemplate` describes how **data is displayed**.

```text
Customer
   ↓
DataTemplate
   ↓
TextBlocks / Images / etc.
```

`ControlTemplate` defines the **visual structure of a control**.

```text
Button
   ↓
ControlTemplate
   ↓
Border + ContentPresenter + ...
```

---

## 10. Custom Controls

### Q24. UserControl vs CustomControl?

Use `UserControl` when composing existing controls into a reusable UI.

Use `CustomControl` when creating a reusable control whose appearance should be defined by themes/templates.

A CustomControl typically uses:

```text
Control class
    +
Generic.xaml
```

This provides stronger template-based customization.

---

## 11. Navigation

### Q25. Should navigation history contain Views or ViewModels?

For MVVM-oriented architecture, storing navigation state around ViewModels is usually preferable.

For example:

```csharp
Stack<object> navigationHistory;
```

where entries represent navigation state/ViewModels.

This avoids tightly coupling navigation logic to concrete Views.

---

### Q26. How do you handle unsaved changes during navigation?

Introduce a navigation guard:

```csharp
public interface INavigationGuard
{
    Task<bool> CanNavigateAwayAsync();
}
```

The navigation service checks the guard before changing screens.

This keeps navigation policy out of the Window/View implementation.

---

## 12. API and Network Architecture

### Q27. Should every ViewModel create its own HttpClient?

No.

Prefer dependency injection and an appropriate `HttpClient` lifetime strategy.

```csharp
public class CustomerService
{
    private readonly HttpClient _client;

    public CustomerService(HttpClient client)
    {
        _client = client;
    }
}
```

The ViewModel depends on the service, not directly on HTTP infrastructure.

---

### Q28. How should retries be designed?

Do not blindly retry everything.

Usually consider retrying transient failures such as:

- temporary network failures
- selected 5xx responses
- throttling, with server guidance

Do not blindly retry:

- validation errors
- authentication failures
- permanent 4xx errors

Use bounded retries and exponential backoff.

---

## 13. Caching

### Q29. How do you prevent a cache stampede?

If many requests miss the same key simultaneously, they can all call the backend.

Use request coalescing / single-flight behavior:

```text
Request A ----\
Request B -----+--> one backend operation --> shared result
Request C ----/
```

A `SemaphoreSlim`, task cache, or dedicated cache abstraction can coordinate concurrent requests.

---

### Q30. How do you invalidate cached data?

Possible strategies:

- TTL
- explicit invalidation
- versioned keys
- event-driven invalidation
- write-through
- refresh-after-write

Choose based on consistency requirements.

---

## 14. Security

### Q31. Is hiding a button a security mechanism?

No.

This:

```csharp
SaveButton.Visibility = Visibility.Collapsed;
```

only changes the UI.

Authorization must be enforced at the actual operation boundary, such as the service/API.

UI authorization improves usability, while backend authorization provides security.

---

### Q32. Where should sensitive credentials be stored?

Do not place secrets in:

```text
appsettings.json
source code
XAML
plain-text configuration
```

Use an appropriate secure credential/token storage mechanism for the environment.

Also avoid logging secrets or access tokens.

---

## 15. Testing

### Q33. What should be tested in a WPF application?

Prioritize:

```text
Business rules
Application services
ViewModel behavior
Commands
Validation
Navigation policies
Error handling
Synchronization logic
```

UI automation should cover important end-to-end workflows rather than every visual detail.

---

### Q34. How do you test a ViewModel?

Inject dependencies:

```csharp
var service = new FakeCustomerService();

var vm = new CustomerViewModel(service);

await vm.LoadAsync();

Assert.Equal("John", vm.Name);
```

Avoid requiring a real Window or real database for a ViewModel unit test.

---

## 16. Legacy WPF Modernization

### Q35. You inherit a 10-year-old WPF application with 500,000 lines of code. What do you do?

Do not immediately rewrite it.

First establish:

```text
Current architecture
Current dependencies
Critical workflows
Performance problems
Production failures
Test coverage
Deployment process
```

Then identify high-value seams for incremental refactoring.

A modernization roadmap might be:

```text
Baseline
  ↓
Characterization tests
  ↓
Extract services
  ↓
Introduce DI
  ↓
Introduce MVVM selectively
  ↓
Modularize
  ↓
Improve observability
  ↓
Replace high-risk legacy components
```

---

## 17. Real-Time WPF Design

### Q36. Design a WPF application receiving 50,000 updates per second.

Never push every network event directly into the UI.

Use a pipeline:

```text
Network
   ↓
Background Receiver
   ↓
Channel<T>
   ↓
Aggregation / Batching
   ↓
State Store
   ↓
UI Dispatcher
   ↓
Virtualized UI
```

The UI should receive an appropriate update rate, not necessarily every raw event.

For example, aggregate changes over a short interval and publish a snapshot/batch.

---

## 18. Offline-First Architecture

### Q37. Design a WPF application that works offline for 8 hours.

Possible architecture:

```text
WPF
 |
Application Services
 |
Local Database
 |
Sync Engine
 |
Network API
```

The sync engine needs:

- local change tracking
- synchronization queue
- retry
- conflict resolution
- idempotency
- authentication refresh
- failure recovery

A critical design question is:

> Which system wins when local and server versions conflict?

The answer depends on business rules.

---

## 19. Large-Scale Modular WPF

### Q38. How would you design a WPF application developed by 20 teams?

Define module boundaries.

```text
Shell
 |
 +-- Customer Module
 +-- Orders Module
 +-- Reporting Module
 +-- Administration Module
 +-- Shared UI
 +-- Shared Application Infrastructure
```

Each module should own its internal Views/ViewModels/services.

Cross-module communication should use explicit contracts rather than direct references wherever practical.

---

## 20. Senior-Level Scenario: CPU at 100%

### Q39. The application reaches 100% CPU after 10 hours. What do you do?

Do not start randomly changing code.

Investigate:

1. CPU profile
2. thread activity
3. timers
4. polling loops
5. runaway tasks
6. retry loops
7. rendering/layout
8. collection processing
9. event storms
10. memory pressure/GC activity

Then reproduce and benchmark the fix.

---

## 21. Senior-Level Scenario: Random UI Freeze

### Q40. Users report random UI freezes. How do you approach it?

Collect:

- timestamp
- operation
- thread state
- application logs
- performance traces
- network/database timing

Common suspects:

```text
UI Thread
   |
   +-- .Result / .Wait()
   +-- synchronous HTTP
   +-- synchronous DB
   +-- large serialization
   +-- expensive calculation
   +-- huge collection update
   +-- layout/rendering storm
```

The answer should demonstrate **diagnosis before optimization**.

---

## 22. Senior-Level Scenario: 1,000 Screens

### Q41. How would you structure an application with 1,000 screens?

Think in modules rather than screens.

```text
Shell
 |
 +-- Customer
 |     +-- Views
 |     +-- ViewModels
 |     +-- Services
 |
 +-- Orders
 |
 +-- Reporting
 |
 +-- Administration
```

Use:

- dependency injection
- navigation abstraction
- shared UI library
- shared infrastructure
- module contracts
- consistent logging
- centralized error handling
- architecture rules

---

## 23. Follow-Up "Why?" Questions

Senior interviewers often keep drilling down.

### Example

**Why MVVM?**

> Separation of concerns and testability.

**Why is separation useful?**

> It reduces coupling between UI and application logic.

**Why does reduced coupling matter?**

> It makes changes easier to isolate and test.

**Why not just use code-behind?**

> Code-behind is perfectly valid for UI-specific behavior, but large amounts of application logic there can make testing and maintenance harder.

The important skill is not memorizing one "correct" architecture. Explain the trade-off.

---

# 24. Rapid-Fire Advanced Questions

1. What is the logical tree?
2. What is the visual tree?
3. What is a DependencyObject?
4. What is DispatcherObject?
5. What is Freezable?
6. Why can a Freezable be shared across threads in certain circumstances?
7. What is a BindingExpression?
8. What is RelativeSource?
9. `ElementName` vs `RelativeSource`?
10. What is `FindAncestor`?
11. What is `TemplateBinding`?
12. `TemplateBinding` vs normal Binding?
13. What is `MultiBinding`?
14. What is `PriorityBinding`?
15. What is a value converter?
16. When should a converter not be used?
17. What is `UpdateSourceTrigger`?
18. Why doesn't TextBox update the source on every keystroke by default?
19. What is `ICollectionView`?
20. What is `CollectionViewSource`?
21. How do filtering and sorting work in WPF collections?
22. What is virtualization?
23. Why can virtualization be accidentally disabled?
24. What is deferred scrolling?
25. What is `IsAsync` in binding?
26. What is a weak event?
27. What is a routed event?
28. Bubbling vs tunneling?
29. What is a class handler?
30. What is command routing?
31. What is an attached behavior?
32. Behavior vs attached property?
33. What is `Adorner`?
34. How would you implement validation with `INotifyDataErrorInfo`?
35. `IDataErrorInfo` vs `INotifyDataErrorInfo`?
36. How do you display validation errors globally?
37. How do you handle localization?
38. How do you support right-to-left layouts?
39. How do you design accessibility?
40. How do you profile rendering performance?

---

# 25. What a Strong Senior Answer Should Demonstrate

For advanced design questions, interviewers are generally looking for more than API knowledge.

A strong answer should cover:

### 1. Requirements

Clarify:

- scale
- concurrency
- latency
- reliability
- offline requirements
- security
- maintainability

### 2. Architecture

Explain:

```text
Components
Responsibilities
Dependencies
Data flow
Threading model
Lifecycle
```

### 3. Failure Modes

Discuss:

- network failure
- cancellation
- timeout
- duplicate operations
- stale data
- application shutdown
- partial failure

### 4. Performance

Explain:

- where work executes
- what happens on the UI thread
- memory usage
- batching
- virtualization
- caching

### 5. Testability

Explain how you would test:

- business logic
- ViewModels
- services
- failure scenarios
- concurrency
- UI workflows

### 6. Trade-offs

This is particularly important at senior level.

Instead of:

> "Always use X."

Prefer:

> "I would use X when ..., but if the requirement is ..., I would consider Y because ..."

---

# 26. Final Advanced Interview Checklist

Before a Senior/Lead WPF interview, make sure you can design and explain:

- [ ] Enterprise WPF architecture
- [ ] MVVM
- [ ] Dependency Injection
- [ ] Modular architecture
- [ ] Navigation
- [ ] Dialog architecture
- [ ] Dependency Properties
- [ ] Data Binding
- [ ] Commands
- [ ] Dispatcher
- [ ] Thread affinity
- [ ] Async/await
- [ ] Cancellation
- [ ] Background processing
- [ ] Channels / producer-consumer
- [ ] UI virtualization
- [ ] Data virtualization
- [ ] DataGrid optimization
- [ ] Memory leaks
- [ ] Event leaks
- [ ] Timers
- [ ] Caching
- [ ] API resilience
- [ ] Offline synchronization
- [ ] Authentication
- [ ] Authorization
- [ ] Secure storage
- [ ] Logging
- [ ] Error handling
- [ ] Testing
- [ ] Legacy modernization
- [ ] Real-time data
- [ ] Large-scale modular applications
- [ ] Performance profiling
- [ ] Architecture trade-offs
- [ ] Production troubleshooting
