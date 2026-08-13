# Chapter 19 — Offline-First WPF Design

## Design question

> Design a WPF application that must work without internet access.

## Architecture

```text
                 ViewModel
                     |
              Application Service
                 /         \
          Local Repository  API
                 \         /
                  Sync Engine
                      |
                 Conflict Rules
```

## Repository abstraction

```csharp
public interface ICustomerRepository
{
    Task<IReadOnlyList<Customer>> GetLocalAsync(
        CancellationToken token);

    Task SaveLocalAsync(
        Customer customer,
        CancellationToken token);
}
```

## Sync record

```csharp
public sealed class SyncRecord
{
    public Guid Id { get; init; }
    public string EntityType { get; init; } = "";
    public string EntityId { get; init; } = "";
    public DateTimeOffset ModifiedAt { get; init; }
    public SyncState State { get; set; }
}

public enum SyncState
{
    Pending,
    Synced,
    Failed
}
```

## Sync workflow

```text
Local change
    |
mark Pending
    |
Application continues working
    |
Connectivity returns
    |
Sync pending changes
    |
Server accepts/rejects
    |
Resolve conflict
    |
Mark Synced
```

## Conflict strategies

Possible strategies:
- server wins
- client wins
- last-write-wins
- merge
- manual conflict resolution

The correct strategy depends on business semantics.

## Important interview point

Offline-first is not simply "cache the API response." You need a local source of truth, change tracking, synchronization, retry behavior, and conflict handling.
