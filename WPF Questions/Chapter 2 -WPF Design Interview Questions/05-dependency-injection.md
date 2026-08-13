# Chapter 5 — Dependency Injection Design

## Design question

> How do you introduce DI into a WPF application?

## Composition root

```csharp
protected override void OnStartup(StartupEventArgs e)
{
    var services = new ServiceCollection();

    services.AddSingleton<IAuthService, AuthService>();
    services.AddSingleton<IApiClient, ApiClient>();
    services.AddSingleton<IDialogService, WpfDialogService>();
    services.AddSingleton<INavigationService, NavigationService>();

    services.AddTransient<MainViewModel>();
    services.AddTransient<CustomerListViewModel>();
    services.AddTransient<CustomerDetailsViewModel>();

    var provider = services.BuildServiceProvider();

    var window = new MainWindow
    {
        DataContext =
            provider.GetRequiredService<MainViewModel>()
    };

    window.Show();
}
```

## Lifetime design

For desktop applications:

- Singleton: application-wide state, configuration, HTTP client abstraction, authentication state.
- Transient: short-lived ViewModels or stateless services.
- Scoped: less common than in web applications; create an explicit scope for a workflow/session if needed.

## Avoid service locator

Bad:

```csharp
var service =
    ServiceLocator.Get<ICustomerService>();
```

Better:

```csharp
public CustomerViewModel(ICustomerService service)
{
    _service = service;
}
```

Dependencies are visible in the constructor.

## Factory for dynamic ViewModels

```csharp
public interface IViewModelFactory
{
    T Create<T>();
}

public sealed class ViewModelFactory : IViewModelFactory
{
    private readonly IServiceProvider _provider;

    public ViewModelFactory(IServiceProvider provider)
    {
        _provider = provider;
    }

    public T Create<T>() =>
        _provider.GetRequiredService<T>();
}
```

## Senior concern

Do not register everything as Singleton. A Singleton that captures a short-lived object can create lifetime bugs and memory leaks.
