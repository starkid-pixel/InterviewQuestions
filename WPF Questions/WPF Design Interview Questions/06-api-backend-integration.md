# Chapter 6 — API / Backend Integration

## Design question

> Design a WPF application that communicates with REST APIs.

## Architecture

```text
View
 |
ViewModel
 |
Application Service
 |
IApiClient
 |
HttpClient
 |
REST API
```

## API client

```csharp
public interface IApiClient
{
    Task<T?> GetAsync<T>(
        string uri,
        CancellationToken cancellationToken);

    Task<TResponse?> PostAsync<TRequest, TResponse>(
        string uri,
        TRequest request,
        CancellationToken cancellationToken);
}
```

## HttpClient implementation

```csharp
public sealed class ApiClient : IApiClient
{
    private readonly HttpClient _http;

    public ApiClient(HttpClient http)
    {
        _http = http;
    }

    public async Task<T?> GetAsync<T>(
        string uri,
        CancellationToken cancellationToken)
    {
        using var response =
            await _http.GetAsync(uri, cancellationToken);

        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<T>(cancellationToken);
    }

    public async Task<TResponse?> PostAsync<TRequest, TResponse>(
        string uri,
        TRequest request,
        CancellationToken cancellationToken)
    {
        using var response =
            await _http.PostAsJsonAsync(
                uri,
                request,
                cancellationToken);

        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<TResponse>(cancellationToken);
    }
}
```

## Application service

```csharp
public interface ICustomerService
{
    Task<IReadOnlyList<CustomerDto>> SearchAsync(
        string query,
        CancellationToken cancellationToken);
}

public sealed class CustomerService : ICustomerService
{
    private readonly IApiClient _api;

    public CustomerService(IApiClient api)
    {
        _api = api;
    }

    public async Task<IReadOnlyList<CustomerDto>> SearchAsync(
        string query,
        CancellationToken cancellationToken)
    {
        var result = await _api.GetAsync<List<CustomerDto>>(
            $"/customers?search={Uri.EscapeDataString(query)}",
            cancellationToken);

        return result ?? [];
    }
}
```

## ViewModel

```csharp
public async Task SearchAsync(CancellationToken token)
{
    try
    {
        Customers =
            await _customerService.SearchAsync(SearchText, token);
    }
    catch (OperationCanceledException)
    {
        // Expected cancellation.
    }
    catch (HttpRequestException ex)
    {
        _logger.LogError(ex, "Customer search failed.");
        ErrorMessage = "Unable to load customers.";
    }
}
```

## Design considerations

Handle:
- timeout
- cancellation
- authentication
- token refresh
- retries for transient failures
- server validation
- logging
- response mapping
- offline behavior

Do not put retry loops and HTTP details in every ViewModel.
