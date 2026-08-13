# Chapter 22 — Testing Architecture

## Design question

> How do you design WPF so that most of the application can be unit tested?

## Testable architecture

```text
View
 |
ViewModel -----> Mock/Fake Services
 |
Application Services -----> Mock API/Repository
 |
Infrastructure
```

## ViewModel test

```csharp
public sealed class FakeCustomerService
    : ICustomerService
{
    public bool Called { get; private set; }

    public Task SaveAsync(
        string name)
    {
        Called = true;
        return Task.CompletedTask;
    }
}
```

```csharp
[Fact]
public async Task Save_Calls_Service()
{
    var service = new FakeCustomerService();

    var vm = new CustomerViewModel(service);

    vm.Name = "Alice";

    await vm.SaveCommand.ExecuteAsync();

    Assert.True(service.Called);
}
```

## Test navigation

```csharp
var navigation = new FakeNavigationService();

var vm = new MainViewModel(navigation);

await vm.OpenCustomerAsync(42);

Assert.Equal(
    42,
    navigation.LastParameter);
```

## Test dialogs

Inject `IDialogService` and provide a fake.

## Test asynchronous operations

Always await the operation:

```csharp
await vm.LoadAsync();

Assert.NotNull(vm.Customers);
```

Avoid tests that rely on arbitrary delays:

```csharp
await Task.Delay(1000); // Bad test synchronization
```

## What should not require a UI test?

Most:
- ViewModel state
- commands
- validation
- service orchestration
- error handling
- navigation requests
- API result mapping

UI automation should focus on actual visual/integration behavior.

## Design principle

Dependency inversion is what makes the architecture testable. If a ViewModel directly constructs `HttpClient`, `Window`, `MessageBox`, database connections, and timers, unit testing becomes unnecessarily difficult.
