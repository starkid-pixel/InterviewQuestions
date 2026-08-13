# Chapter 14 — Authentication and Authorization

## Design question

> Design authentication and authorization for a WPF application communicating with a REST API.

## Architecture

```text
Login View
    |
Login ViewModel
    |
IAuthenticationService
    |
Token / Identity Store
    |
API Client
```

## Authentication contract

```csharp
public interface IAuthenticationService
{
    Task<AuthenticationResult> LoginAsync(
        string username,
        string password,
        CancellationToken token);

    Task LogoutAsync();

    bool IsAuthenticated { get; }

    UserIdentity? CurrentUser { get; }
}
```

## API authorization

The API client should attach access credentials centrally rather than requiring every ViewModel to do it.

```csharp
public sealed class AuthenticatedApiClient
{
    private readonly HttpClient _http;
    private readonly ITokenStore _tokens;

    public AuthenticatedApiClient(
        HttpClient http,
        ITokenStore tokens)
    {
        _http = http;
        _tokens = tokens;
    }

    public async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken token)
    {
        var accessToken = await _tokens.GetAccessTokenAsync();

        request.Headers.Authorization =
            new AuthenticationHeaderValue(
                "Bearer",
                accessToken);

        return await _http.SendAsync(request, token);
    }
}
```

## Token expiration

Centralize handling:
1. detect 401
2. attempt refresh when supported
3. retry the original request safely
4. if refresh fails, transition to logged-out state

Avoid infinite retry loops.

## Authorization

UI visibility is convenience, not security.

```csharp
public bool CanEditCustomers =>
    CurrentUser?.Permissions.Contains("Customer.Edit") == true;
```

The server must enforce authorization.

## Logout

Logout should:
- clear credentials securely
- cancel sensitive ongoing operations
- clear user-specific cache
- reset navigation/state
- return to login
