# Chapter 3 — Navigation Design

## Design question

> Design navigation for a WPF application with 100+ screens without coupling ViewModels to concrete Views.

## Architecture

```text
ShellViewModel
       |
       v
INavigationService
       |
       v
NavigationService
       |
       +--> ViewModel factory
       +--> navigation history
       +--> parameter passing
       +--> lifetime management
```

## Navigation contract

```csharp
public interface INavigationService
{
    Task NavigateAsync<TViewModel>(object? parameter = null);
    bool CanGoBack { get; }
    Task GoBackAsync();
}
```

## Navigation service

```csharp
public sealed class NavigationService : INavigationService
{
    private readonly IServiceProvider _services;
    private readonly Stack<object> _history = new();

    public NavigationService(IServiceProvider services)
    {
        _services = services;
    }

    public object? Current { get; private set; }

    public bool CanGoBack => _history.Count > 0;

    public Task NavigateAsync<TViewModel>(object? parameter = null)
    {
        if (Current is not null)
            _history.Push(Current);

        var vm = _services.GetRequiredService<TViewModel>();

        if (vm is INavigationAware aware)
            aware.OnNavigatedTo(parameter);

        Current = vm;

        return Task.CompletedTask;
    }

    public Task GoBackAsync()
    {
        if (_history.Count == 0)
            return Task.CompletedTask;

        if (Current is INavigationAware oldAware)
            oldAware.OnNavigatedFrom();

        Current = _history.Pop();

        return Task.CompletedTask;
    }
}
```

## Navigation-aware contract

```csharp
public interface INavigationAware
{
    void OnNavigatedTo(object? parameter);
    void OnNavigatedFrom();
}
```

For asynchronous loading:

```csharp
public interface IAsyncNavigationAware
{
    Task OnNavigatedToAsync(object? parameter);
    Task OnNavigatedFromAsync();
}
```

## Parameter example

```csharp
public record CustomerDetailsParameter(int CustomerId);

public sealed class CustomerDetailsViewModel
    : IAsyncNavigationAware
{
    public Task OnNavigatedToAsync(object? parameter)
    {
        var p = (CustomerDetailsParameter)parameter!;
        return LoadCustomerAsync(p.CustomerId);
    }

    public Task OnNavigatedFromAsync()
    {
        return Task.CompletedTask;
    }

    private Task LoadCustomerAsync(int id)
    {
        // Load customer.
        return Task.CompletedTask;
    }
}
```

## Navigation history

Be careful about storing entire ViewModels indefinitely. For large applications, history may store navigation entries instead:

```csharp
public record NavigationEntry(
    Type ViewModelType,
    object? Parameter);
```

This allows recreation rather than retaining large object graphs.

## Interview follow-up

**How do you avoid memory leaks?**

Define a lifecycle:
- cancel ongoing work on navigation away
- unsubscribe event handlers
- stop timers
- release subscriptions
- avoid global publishers retaining short-lived ViewModels
- decide explicitly whether history retains or recreates ViewModels
