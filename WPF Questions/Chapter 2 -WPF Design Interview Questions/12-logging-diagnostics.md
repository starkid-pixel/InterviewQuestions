# Chapter 12 — Logging and Diagnostics

## Design question

> How would you design production logging for a WPF application?

## Architecture

```text
ViewModel / Service
        |
      ILogger
        |
 Logging Provider
        |
File / Windows Event Log / Centralized System
```

## Example

```csharp
public sealed class CustomerService
{
    private readonly ILogger<CustomerService> _logger;

    public CustomerService(
        ILogger<CustomerService> logger)
    {
        _logger = logger;
    }

    public async Task<CustomerDto?> GetAsync(
        int id,
        CancellationToken token)
    {
        try
        {
            _logger.LogInformation(
                "Loading customer {CustomerId}",
                id);

            return await LoadCoreAsync(id, token);
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Failed to load customer {CustomerId}",
                id);

            throw;
        }
    }

    private Task<CustomerDto?> LoadCoreAsync(
        int id,
        CancellationToken token)
    {
        return Task.FromResult<CustomerDto?>(null);
    }
}
```

## Global exception boundaries

```csharp
public partial class App : Application
{
    protected override void OnStartup(
        StartupEventArgs e)
    {
        DispatcherUnhandledException +=
            OnDispatcherUnhandledException;

        TaskScheduler.UnobservedTaskException +=
            OnUnobservedTaskException;

        base.OnStartup(e);
    }

    private void OnDispatcherUnhandledException(
        object sender,
        DispatcherUnhandledExceptionEventArgs e)
    {
        // Log centrally.
        e.Handled = true;
    }

    private void OnUnobservedTaskException(
        object? sender,
        UnobservedTaskExceptionEventArgs e)
    {
        // Log centrally.
        e.SetObserved();
    }
}
```

Global handlers are safety nets, not substitutes for handling expected failures close to their source.

## Production diagnostics

Capture:
- correlation/request IDs
- operation names
- timing
- exceptions
- user-safe context
- API endpoint categories
- application version

Never log passwords, tokens, or sensitive secrets.
