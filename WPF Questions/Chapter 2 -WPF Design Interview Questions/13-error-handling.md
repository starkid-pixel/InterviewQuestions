# Chapter 13 — Error Handling Design

## Design question

> Design centralized error handling for a WPF application.

## Classify failures

```text
                Failure
                   |
        +----------+----------+
        |                     |
 Expected                 Unexpected
        |                     |
Business/API             Exception
 validation                  |
        |                 Log + diagnostics
User-friendly                 |
message                    Safe fallback
```

## Result type

For expected application failures, a Result pattern can be useful:

```csharp
public sealed record Result<T>(
    bool Success,
    T? Value,
    string? Error)
{
    public static Result<T> Ok(T value)
        => new(true, value, null);

    public static Result<T> Fail(string error)
        => new(false, default, error);
}
```

## Service

```csharp
public async Task<Result<CustomerDto>> SaveAsync(
    CustomerDto customer,
    CancellationToken token)
{
    try
    {
        var saved = await _api.SaveAsync(customer, token);
        return Result<CustomerDto>.Ok(saved);
    }
    catch (ValidationException ex)
    {
        return Result<CustomerDto>.Fail(ex.Message);
    }
}
```

## ViewModel

```csharp
var result = await _service.SaveAsync(Customer, token);

if (!result.Success)
{
    ErrorMessage = result.Error;
    return;
}

Customer = result.Value!;
```

## Unexpected exceptions

Do not swallow:

```csharp
catch (Exception)
{
    // Nothing
}
```

Instead log and either rethrow or convert to an appropriate application-level result.

## Retry policy

Retry only failures likely to succeed later, such as transient network failures. Do not blindly retry validation failures or authorization failures.

## Interview follow-up

> Where should errors be displayed?

The layer that understands user interaction should generally decide how to present an error. Infrastructure should report technical details; application services can classify failures; ViewModels can translate them into presentation state.
