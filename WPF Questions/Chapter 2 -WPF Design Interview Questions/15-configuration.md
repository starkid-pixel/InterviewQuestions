# Chapter 15 — Configuration Design

## Design question

> How do you manage configuration across development, test, and production?

## Configuration abstraction

```csharp
public sealed class ApiOptions
{
    public string BaseUrl { get; set; } = "";
    public TimeSpan Timeout { get; set; }
        = TimeSpan.FromSeconds(30);
}
```

Register:

```csharp
services.Configure<ApiOptions>(
    configuration.GetSection("Api"));
```

Consume:

```csharp
public sealed class ApiClient
{
    private readonly ApiOptions _options;

    public ApiClient(
        IOptions<ApiOptions> options)
    {
        _options = options.Value;
    }
}
```

## Example configuration

```json
{
  "Api": {
    "BaseUrl": "https://api.example.com",
    "Timeout": "00:00:30"
  }
}
```

## Environment separation

Do not hard-code:

```csharp
new HttpClient
{
    BaseAddress =
        new Uri("https://production.example.com")
};
```

Prefer environment-specific configuration.

## Secrets

Do not put passwords, private keys, or long-lived secrets in source-controlled configuration.

Use an appropriate OS/platform secret store or enterprise credential mechanism.

## Runtime configuration

If settings can change while the application is running, define explicit behavior:
- reload automatically
- apply to new operations
- notify existing components
- require restart

Do not make mutable global configuration silently change behavior everywhere.
