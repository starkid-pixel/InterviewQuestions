# Chapter 18 — Plugin / Modular Architecture

## Design question

> Design a WPF application where different teams can build independent feature modules.

## Architecture

```text
                    Shell
                      |
        +-------------+-------------+
        |             |             |
    Customers       Orders       Reports
      Module         Module        Module
        |             |             |
      Views          Views         Views
      VMs            VMs           VMs
        |             |             |
    Services        Services      Services
```

## Module contract

```csharp
public interface IModule
{
    string Name { get; }

    Task InitializeAsync(
        IServiceCollection services);
}
```

A more mature implementation usually separates registration from initialization:

```csharp
public interface IModule
{
    string Name { get; }

    void RegisterServices(IServiceCollection services);

    Task InitializeAsync(IServiceProvider provider);
}
```

## Module registration

```csharp
public sealed class CustomersModule : IModule
{
    public string Name => "Customers";

    public void RegisterServices(
        IServiceCollection services)
    {
        services.AddTransient<CustomerListViewModel>();
        services.AddTransient<CustomerDetailsViewModel>();
        services.AddSingleton<ICustomerService,
            CustomerService>();
    }

    public Task InitializeAsync(
        IServiceProvider provider)
    {
        return Task.CompletedTask;
    }
}
```

## Communication between modules

Prefer shared contracts:

```csharp
public record CustomerSelectedMessage(int CustomerId);
```

Publish/subscribe can be used when direct coupling would otherwise be excessive.

## Plugin loading

A plugin system may use `AssemblyLoadContext` to load assemblies dynamically:

```csharp
var assembly =
    AssemblyLoadContext.Default.LoadFromAssemblyPath(path);

var modules =
    assembly.GetTypes()
        .Where(t => typeof(IModule).IsAssignableFrom(t)
                 && !t.IsAbstract)
        .Select(t => (IModule)Activator.CreateInstance(t)!);
```

For true unloadable plugins, use a collectible `AssemblyLoadContext` and carefully control references.

## Key design risks

- version compatibility
- dependency conflicts
- unloadability
- security
- shared contracts
- module lifecycle
- global event leaks

Do not let modules directly manipulate each other's Views.
