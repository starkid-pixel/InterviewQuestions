# Chapter 10 — Caching Design

## Design question

> How would you design caching in a WPF application?

## Layers

```text
ViewModel
   |
Application Service
   |
Cache
   |
API / Repository
```

## Cache abstraction

```csharp
public interface IAppCache
{
    bool TryGet<T>(string key, out T? value);

    void Set<T>(
        string key,
        T value,
        TimeSpan lifetime);

    void Remove(string key);
}
```

## Simple memory cache

```csharp
public sealed class AppCache : IAppCache
{
    private readonly MemoryCache _cache =
        new(new MemoryCacheOptions());

    public bool TryGet<T>(
        string key,
        out T? value)
    {
        if (_cache.TryGetValue(key, out var obj) &&
            obj is T typed)
        {
            value = typed;
            return true;
        }

        value = default;
        return false;
    }

    public void Set<T>(
        string key,
        T value,
        TimeSpan lifetime)
    {
        _cache.Set(key, value, lifetime);
    }

    public void Remove(string key)
        => _cache.Remove(key);
}
```

## Service

```csharp
public async Task<CustomerDto?> GetAsync(
    int id,
    CancellationToken token)
{
    var key = $"customer:{id}";

    if (_cache.TryGet<CustomerDto>(key, out var cached))
        return cached;

    var customer =
        await _api.GetAsync<CustomerDto>(
            $"/customers/{id}",
            token);

    if (customer is not null)
        _cache.Set(key, customer, TimeSpan.FromMinutes(5));

    return customer;
}
```

## Cache questions

Always define:
- expiration
- invalidation
- maximum size
- stale data policy
- concurrency behavior
- persistence requirements

For desktop applications, memory caching is useful for session-lifetime data. Disk caching is appropriate when offline behavior or startup performance requires persistence.
