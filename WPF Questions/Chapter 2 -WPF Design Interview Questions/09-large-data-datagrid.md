# Chapter 9 — Large Data / DataGrid Design

## Design question

> Design a WPF screen capable of displaying millions of records.

## Wrong architecture

```text
Database
   |
1,000,000 rows
   |
1,000,000 objects
   |
ObservableCollection
   |
DataGrid
```

This wastes memory and makes filtering/sorting expensive.

## Better architecture

```text
DataGrid
   |
ViewModel
   |
Paged / Virtualized Data Provider
   |
Application Service
   |
API
   |
Database
```

## Page model

```csharp
public sealed record PageRequest(
    int PageNumber,
    int PageSize,
    string? Search,
    string? SortColumn,
    bool Descending);

public sealed record PageResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount);
```

## Service

```csharp
public interface ICustomerSearchService
{
    Task<PageResult<CustomerDto>> SearchAsync(
        PageRequest request,
        CancellationToken cancellationToken);
}
```

## ViewModel

```csharp
public sealed class CustomerGridViewModel
{
    private readonly ICustomerSearchService _service;

    public ObservableCollection<CustomerDto> Items { get; } = [];

    public int PageSize { get; set; } = 100;

    public async Task LoadPageAsync(
        int page,
        CancellationToken token)
    {
        var result = await _service.SearchAsync(
            new PageRequest(
                page,
                PageSize,
                SearchText,
                SortColumn,
                SortDescending),
            token);

        Items.Clear();

        foreach (var item in result.Items)
            Items.Add(item);

        TotalCount = result.TotalCount;
    }
}
```

## XAML virtualization

```xml
<DataGrid
    ItemsSource="{Binding Items}"
    EnableRowVirtualization="True"
    EnableColumnVirtualization="True"
    VirtualizingPanel.IsVirtualizing="True"
    VirtualizingPanel.VirtualizationMode="Recycling"
    ScrollViewer.CanContentScroll="True"
    AutoGenerateColumns="False" />
```

## Design choices

Use:
- server-side paging
- server-side filtering
- server-side sorting
- UI virtualization
- lightweight row templates
- cancellation of obsolete searches
- debouncing search input

## Search cancellation

When a new search starts, cancel the previous one:

```csharp
private CancellationTokenSource? _searchCts;

public async Task SearchAsync()
{
    _searchCts?.Cancel();
    _searchCts?.Dispose();

    _searchCts = new CancellationTokenSource();

    await LoadPageAsync(
        1,
        _searchCts.Token);
}
```

## Interview point

UI virtualization solves UI-object explosion. It does not solve loading millions of database records into memory. Large-data architecture normally needs data virtualization/paging as well.
