# Chapter 8 — Thread-Safe UI Design

## Design question

> A background service receives data and must update an ObservableCollection displayed by WPF. Design it safely.

## Problem

```csharp
// Potentially unsafe if executed on worker thread:
Customers.Add(customer);
```

A UI-bound collection may need modification on the UI thread.

## Dispatcher abstraction

```csharp
public interface IUiDispatcher
{
    Task InvokeAsync(Action action);
}
```

## WPF implementation

```csharp
public sealed class WpfUiDispatcher : IUiDispatcher
{
    private readonly Dispatcher _dispatcher;

    public WpfUiDispatcher(Dispatcher dispatcher)
    {
        _dispatcher = dispatcher;
    }

    public Task InvokeAsync(Action action)
    {
        return _dispatcher.InvokeAsync(action).Task;
    }
}
```

## ViewModel

```csharp
private readonly IUiDispatcher _dispatcher;

public ObservableCollection<Customer> Customers { get; } = [];

public async Task AddCustomerAsync(Customer customer)
{
    await _dispatcher.InvokeAsync(
        () => Customers.Add(customer));
}
```

## Better architecture for high-frequency updates

Do not dispatch thousands of individual UI updates.

Batch them:

```csharp
public async Task AddBatchAsync(
    IReadOnlyList<Customer> customers)
{
    await _dispatcher.InvokeAsync(() =>
    {
        foreach (var customer in customers)
            Customers.Add(customer);
    });
}
```

For very high rates, consider:
- batching
- throttling
- producer/consumer channels
- collection synchronization
- UI virtualization

## Where should Dispatcher logic live?

Prefer infrastructure/application coordination rather than scattering:

```csharp
Application.Current.Dispatcher.Invoke(...)
```

through every ViewModel. An abstraction makes threading policy testable and centralized.
