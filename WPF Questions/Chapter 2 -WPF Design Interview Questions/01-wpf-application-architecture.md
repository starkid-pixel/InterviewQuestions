# Chapter 1 — WPF Application Architecture

## 1. Design question

> Design a large WPF enterprise application using MVVM, REST APIs, authentication, logging, navigation, dialogs, background processing, and multiple teams.

## Recommended architecture

```text
+------------------------------------------------------+
|                       WPF Shell                      |
|                                                      |
|  Views / XAML                                       |
|       |                                              |
|  ViewModels / Commands                              |
|       |                                              |
|  Application Services                               |
|       |                                              |
|  Domain / Business Rules                            |
|       |                                              |
|  Infrastructure                                     |
|   +---- API Client                                   |
|   +---- Persistence                                  |
|   +---- Authentication                               |
|   +---- Logging                                      |
|   +---- Configuration                                |
|                                                      |
+------------------------------------------------------+
```

A practical solution separates presentation, application orchestration, domain rules, and infrastructure.

## Suggested project structure

```text
MyApp.sln
 |
 +-- MyApp.Desktop
 |    +-- App.xaml
 |    +-- Shell
 |    +-- Views
 |    +-- ViewModels
 |    +-- Controls
 |    +-- Resources
 |
 +-- MyApp.Application
 |    +-- Services
 |    +-- Commands
 |    +-- DTOs
 |    +-- Interfaces
 |
 +-- MyApp.Domain
 |    +-- Entities
 |    +-- Rules
 |    +-- ValueObjects
 |
 +-- MyApp.Infrastructure
      +-- Api
      +-- Persistence
      +-- Authentication
      +-- Logging
      +-- Configuration
```

## Dependency direction

```text
Desktop --> Application --> Domain
Desktop --> Infrastructure
Infrastructure --> Application
Infrastructure --> Domain
```

The Domain should not depend on WPF.

## Composition root

```csharp
public partial class App : Application
{
    private IServiceProvider _services = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var services = new ServiceCollection();

        services.AddSingleton<IAuthService, AuthService>();
        services.AddSingleton<ICustomerService, CustomerService>();
        services.AddSingleton<INavigationService, NavigationService>();

        services.AddTransient<MainViewModel>();
        services.AddTransient<CustomerListViewModel>();

        _services = services.BuildServiceProvider();

        var shell = new MainWindow
        {
            DataContext = _services.GetRequiredService<MainViewModel>()
        };

        shell.Show();
    }
}
```

## Main design principle

The View should be responsible primarily for presentation. The ViewModel should own presentation state and commands. Application services should orchestrate use cases. Infrastructure should handle external systems.

## What not to do

```csharp
public class CustomerViewModel
{
    public async Task Load()
    {
        using var client = new HttpClient();
        var response = await client.GetAsync("https://api.example.com/customers");

        // Bad: ViewModel now owns HTTP infrastructure.
    }
}
```

Prefer:

```csharp
public class CustomerViewModel
{
    private readonly ICustomerService _service;

    public CustomerViewModel(ICustomerService service)
    {
        _service = service;
    }

    public async Task LoadAsync()
    {
        Customers = await _service.GetCustomersAsync();
    }
}
```

## Senior follow-up

**How do you scale this to multiple teams?**

Use feature/module boundaries, stable contracts, dependency inversion, shared UI infrastructure, coding conventions, automated tests, and clear ownership of cross-cutting services.

---

# Key interview answer

> "I would keep WPF-specific concerns in the presentation layer, application use cases behind interfaces, domain rules independent of WPF, and infrastructure behind abstractions. The composition root wires everything together. This gives us testability, replaceability, and modularity."
