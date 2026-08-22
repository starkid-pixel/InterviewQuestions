Performance Tactics Demo — Complete Source

A single WPF application demonstrating the Control Resource Demand tactics.

App.xaml

<Application x:Class="PerformanceTacticsDemo.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources />
</Application>

App.xaml.cs

using System.Windows;

namespace PerformanceTacticsDemo;

public partial class App : Application
{
}

Docs/README.md

# Control Resource Demand — WPF Performance Demo

This demo uses one Product Monitoring Dashboard to demonstrate six
**Control Resource Demand** tactics through eight concrete scenarios.

## Tactic mapping

| Scenario | Tactic | Implementation |
|---|---|---|
| Search | Limit Event Response | Debouncing |
| Live updates | Manage Sampling Rate | Throttling / latest-value sampling |
| Incoming updates | Limit Event Response + Reduce Overhead | Event coalescing + batching |
| Alerts | Prioritize Events | Priority Queue |
| Product grid | Reduce Overhead | Paging |
| Product grid UI | Resource efficiency | UI virtualization |
| Product details | Reduce Overhead | Caching |
| Slow operation | Bound Execution Times | Timeout + cancellation |
| Export | Increase Resource Efficiency | Streaming + chunking + concurrency control |

## Architecture

```text
Performance Efficiency
        ↓
Control Resource Demand
        ↓
Tactics
        ↓
Concrete approaches
        ↓
Technology implementation
        ↓
WPF UI

Scenarios

Search + Debouncing

Live Updates + Throttling

Batching + Event Coalescing

Priority Queue

Paging + UI Virtualization

Product Detail Caching

Timeout + Cancellation

Export + Streaming + Chunking + Concurrency Control

The project intentionally uses fake/in-memory data so the performance
tactics remain easy to observe without adding backend infrastructure.


## `Docs/control-resource-demand-architecture.svg`

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     width="1200" height="760" viewBox="0 0 1200 760">
  <rect width="1200" height="760" fill="white"/>
  <style>
    .box { fill:white; stroke:#333; stroke-width:2; rx:10; }
    .title { font:bold 26px sans-serif; fill:#111; }
    .text { font:bold 17px sans-serif; fill:#111; }
    .small { font:15px sans-serif; fill:#333; }
    .line { stroke:#333; stroke-width:2; fill:none; }
  </style>

  <text x="600" y="42" text-anchor="middle" class="title">
    Control Resource Demand — WPF Performance Demo
  </text>

  <rect x="400" y="70" width="400" height="65" class="box"/>
  <text x="600" y="110" text-anchor="middle" class="text">
    Product Service / Incoming Events
  </text>

  <line x1="600" y1="135" x2="600" y2="180" class="line"/>

  <rect x="360" y="180" width="480" height="70" class="box"/>
  <text x="600" y="210" text-anchor="middle" class="text">
    Control Resource Demand
  </text>
  <text x="600" y="233" text-anchor="middle" class="small">
    Performance Efficiency
  </text>

  <rect x="40" y="300" width="210" height="115" class="box"/>
  <text x="145" y="332" text-anchor="middle" class="text">Sampling Rate</text>
  <text x="145" y="360" text-anchor="middle" class="small">Throttling</text>
  <text x="145" y="385" text-anchor="middle" class="small">Latest value</text>

  <rect x="270" y="300" width="210" height="115" class="box"/>
  <text x="375" y="332" text-anchor="middle" class="text">Event Response</text>
  <text x="375" y="360" text-anchor="middle" class="small">Debouncing</text>
  <text x="375" y="385" text-anchor="middle" class="small">Coalescing</text>

  <rect x="500" y="300" width="210" height="115" class="box"/>
  <text x="605" y="332" text-anchor="middle" class="text">Prioritize Events</text>
  <text x="605" y="360" text-anchor="middle" class="small">Priority Queue</text>
  <text x="605" y="385" text-anchor="middle" class="small">Critical work first</text>

  <rect x="730" y="300" width="210" height="115" class="box"/>
  <text x="835" y="332" text-anchor="middle" class="text">Reduce Overhead</text>
  <text x="835" y="360" text-anchor="middle" class="small">Batch / Cache / Paging</text>
  <text x="835" y="385" text-anchor="middle" class="small">Avoid duplicate work</text>

  <rect x="960" y="300" width="200" height="115" class="box"/>
  <text x="1060" y="332" text-anchor="middle" class="text">Bound Execution</text>
  <text x="1060" y="360" text-anchor="middle" class="small">Timeout</text>
  <text x="1060" y="385" text-anchor="middle" class="small">Cancellation</text>

  <line x1="600" y1="250" x2="145" y2="300" class="line"/>
  <line x1="600" y1="250" x2="375" y2="300" class="line"/>
  <line x1="600" y1="250" x2="605" y2="300" class="line"/>
  <line x1="600" y1="250" x2="835" y2="300" class="line"/>
  <line x1="600" y1="250" x2="1060" y2="300" class="line"/>

  <rect x="350" y="480" width="500" height="105" class="box"/>
  <text x="600" y="515" text-anchor="middle" class="text">
    Increase Resource Efficiency
  </text>
  <text x="600" y="545" text-anchor="middle" class="small">
    Streaming · Chunking · Concurrency Control
  </text>
  <text x="600" y="570" text-anchor="middle" class="small">
    Export scenario
  </text>

  <line x1="605" y1="415" x2="600" y2="480" class="line"/>

  <rect x="420" y="635" width="360" height="65" class="box"/>
  <text x="600" y="675" text-anchor="middle" class="text">
    ViewModels → WPF UI
  </text>

  <line x1="600" y1="585" x2="600" y2="635" class="line"/>
</svg>

Infrastructure/Debouncer.cs

namespace PerformanceTacticsDemo.Infrastructure;

public sealed class Debouncer
{
    private CancellationTokenSource? _cts;

    public async Task ExecuteAsync(
        TimeSpan delay,
        Func<CancellationToken, Task> action)
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource();

        try
        {
            await Task.Delay(delay, _cts.Token);
            await action(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            // Expected when a newer event replaces this operation.
        }
    }
}

Infrastructure/ObservableObject.cs

using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace PerformanceTacticsDemo.Infrastructure;

public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged(
        [CallerMemberName] string? propertyName = null) =>
        PropertyChanged?.Invoke(
            this,
            new PropertyChangedEventArgs(propertyName));

    protected bool SetProperty<T>(
        ref T field,
        T value,
        [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;

        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
}

Infrastructure/PriorityWorkQueue.cs

namespace PerformanceTacticsDemo.Infrastructure;

public class PriorityWorkQueue<T>
{
    private readonly PriorityQueue<T, int> _queue = new();

    public void Enqueue(T item, int priority) =>
        _queue.Enqueue(item, priority);

    public bool TryDequeue(out T? item) =>
        _queue.TryDequeue(out item, out _);

    public int Count => _queue.Count;
}

Infrastructure/RelayCommand.cs

using System.Windows.Input;

namespace PerformanceTacticsDemo.Infrastructure;

public class RelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool>? _canExecute;

    public RelayCommand(
        Action execute,
        Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public event EventHandler? CanExecuteChanged;

    public bool CanExecute(object? parameter) =>
        _canExecute?.Invoke() ?? true;

    public void Execute(object? parameter) =>
        _execute();

    public void RaiseCanExecuteChanged() =>
        CanExecuteChanged?.Invoke(
            this,
            EventArgs.Empty);
}

MainWindow.xaml

<Window x:Class="PerformanceTacticsDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:views="clr-namespace:PerformanceTacticsDemo.Views"
        Title="Control Resource Demand - WPF Demo"
        Height="700"
        Width="1100">

    <Grid Margin="20">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition/>
        </Grid.RowDefinitions>

        <TextBlock Text="Control Resource Demand - WPF Demo"
                   FontSize="24"
                   FontWeight="Bold"
                   Margin="0,0,0,15"/>

        <TabControl Grid.Row="1">
            <TabItem Header="1. Debounce">
                <views:SearchView/>
            </TabItem>

            <TabItem Header="2. Throttle">
                <views:LiveUpdatesView/>
            </TabItem>

            <TabItem Header="3. Batch + Coalesce">
                <views:BatchUpdatesView/>
            </TabItem>

            <TabItem Header="4. Priority">
                <views:PriorityAlertsView/>
            </TabItem>

            <TabItem Header="5. Paging">
                <views:PagingView/>
            </TabItem>

            <TabItem Header="6. Cache">
                <views:CacheView/>
            </TabItem>

            <TabItem Header="7. Timeout + Cancellation">
                <views:TimeoutView/>
            </TabItem>

            <TabItem Header="8. Export">
                <views:ExportView/>
            </TabItem>
        </TabControl>
    </Grid>
</Window>

MainWindow.xaml.cs

using System.Windows;
using PerformanceTacticsDemo.Services;
using PerformanceTacticsDemo.ViewModels;

namespace PerformanceTacticsDemo;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        var productService = new ProductService();
        var productCache = new ProductCache();
        var exportService = new ExportService();

        DataContext = new MainViewModel(
            productService,
            productCache,
            exportService);
    }
}

Models/Alert.cs

namespace PerformanceTacticsDemo.Models;

public enum AlertPriority
{
    Critical = 0,
    High = 1,
    Normal = 2,
    Low = 3
}

public class Alert
{
    public string Message { get; init; } = string.Empty;
    public AlertPriority Priority { get; init; }

    public override string ToString() =>
        $"[{Priority}] {Message}";
}

Models/Product.cs

namespace PerformanceTacticsDemo.Models;

public class Product
{
    public int Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public string Category { get; init; } = string.Empty;

    public override string ToString() =>
        $"{Id} - {Name} - {Price:C}";
}

Models/ProductUpdate.cs

namespace PerformanceTacticsDemo.Models;

public class ProductUpdate
{
    public int ProductId { get; init; }
    public decimal NewPrice { get; init; }
    public DateTime Timestamp { get; init; }
}

PerformanceTacticsDemo.csproj

<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>

Services/ExportService.cs

using PerformanceTacticsDemo.Models;

namespace PerformanceTacticsDemo.Services;

public class ExportService
{
    public async Task ExportAsync(
        IAsyncEnumerable<Product> products,
        int chunkSize,
        int maxConcurrency,
        IProgress<int>? progress = null,
        CancellationToken cancellationToken = default)
    {
        var buffer = new List<Product>();
        var processed = 0;

        using var semaphore =
            new SemaphoreSlim(maxConcurrency);

        var tasks = new List<Task>();

        async Task ProcessChunkAsync(
            List<Product> chunk)
        {
            await semaphore.WaitAsync(
                cancellationToken);

            try
            {
                await ExportChunkAsync(
                    chunk,
                    cancellationToken);

                var current = Interlocked.Add(
                    ref processed,
                    chunk.Count);

                progress?.Report(current);
            }
            finally
            {
                semaphore.Release();
            }
        }

        await foreach (
            var product in products.WithCancellation(
                cancellationToken))
        {
            buffer.Add(product);

            if (buffer.Count < chunkSize)
                continue;

            var chunk = buffer.ToList();
            buffer.Clear();

            tasks.Add(ProcessChunkAsync(chunk));
        }

        if (buffer.Count > 0)
            tasks.Add(ProcessChunkAsync(
                buffer.ToList()));

        await Task.WhenAll(tasks);
    }

    private static async Task ExportChunkAsync(
        List<Product> products,
        CancellationToken cancellationToken)
    {
        // Simulates writing a chunk to a file/service.
        await Task.Delay(50, cancellationToken);
    }
}

Services/ProductCache.cs

using PerformanceTacticsDemo.Models;

namespace PerformanceTacticsDemo.Services;

public class ProductCache
{
    private readonly Dictionary<int, Product> _cache = new();

    public bool TryGet(
        int id,
        out Product? product) =>
        _cache.TryGetValue(id, out product);

    public void Set(Product product) =>
        _cache[product.Id] = product;
}

Services/ProductService.cs

using PerformanceTacticsDemo.Models;

namespace PerformanceTacticsDemo.Services;

public class ProductService
{
    private readonly List<Product> _products;

    public ProductService()
    {
        _products = Enumerable
            .Range(1, 100_000)
            .Select(i => new Product
            {
                Id = i,
                Name = $"Product {i}",
                Price = Random.Shared.Next(10, 10_000),
                Category = $"Category {i % 10}"
            })
            .ToList();
    }

    public async Task<List<Product>> SearchAsync(
        string searchText,
        CancellationToken cancellationToken = default)
    {
        await Task.Delay(300, cancellationToken);

        return _products
            .Where(p => p.Name.Contains(
                searchText,
                StringComparison.OrdinalIgnoreCase))
            .Take(100)
            .ToList();
    }

    public async Task<Product?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken = default)
    {
        await Task.Delay(300, cancellationToken);
        return _products.FirstOrDefault(x => x.Id == id);
    }

    public async Task<List<Product>> GetPageAsync(
        int pageNumber,
        int pageSize)
    {
        await Task.Delay(100);

        return _products
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToList();
    }

    public int TotalCount => _products.Count;

    public async IAsyncEnumerable<Product> StreamAllAsync()
    {
        foreach (var product in _products)
        {
            await Task.Delay(1);
            yield return product;
        }
    }
}

ViewModels/BatchUpdatesViewModel.cs

using System.Collections.ObjectModel;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Models;

namespace PerformanceTacticsDemo.ViewModels;

public class BatchUpdatesViewModel : ObservableObject
{
    private readonly Dictionary<int, ProductUpdate>
        _latestUpdates = new();

    private readonly PeriodicTimer _timer =
        new(TimeSpan.FromMilliseconds(500));

    public ObservableCollection<string>
        ProcessedUpdates { get; } = new();

    public BatchUpdatesViewModel()
    {
        _ = StartBatchProcessorAsync();
    }

    public void ReceiveUpdate(
        ProductUpdate update)
    {
        lock (_latestUpdates)
        {
            // Event coalescing:
            // keep only the latest update per product.
            _latestUpdates[update.ProductId] = update;
        }
    }

    private async Task StartBatchProcessorAsync()
    {
        while (await _timer.WaitForNextTickAsync())
        {
            List<ProductUpdate> batch;

            lock (_latestUpdates)
            {
                batch = _latestUpdates.Values.ToList();
                _latestUpdates.Clear();
            }

            foreach (var update in batch)
            {
                ProcessedUpdates.Add(
                    $"Product {update.ProductId} -> " +
                    $"{update.NewPrice}");
            }
        }
    }
}

ViewModels/CacheViewModel.cs

using System.Windows.Input;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Models;
using PerformanceTacticsDemo.Services;

namespace PerformanceTacticsDemo.ViewModels;

public class CacheViewModel : ObservableObject
{
    private readonly ProductService _productService;
    private readonly ProductCache _cache;

    private int _productId = 1;
    private Product? _product;
    private string _source = string.Empty;

    public int ProductId
    {
        get => _productId;
        set => SetProperty(
            ref _productId,
            value);
    }

    public Product? Product
    {
        get => _product;
        set => SetProperty(
            ref _product,
            value);
    }

    public string Source
    {
        get => _source;
        set => SetProperty(
            ref _source,
            value);
    }

    public ICommand LoadCommand { get; }

    public CacheViewModel(
        ProductService productService,
        ProductCache cache)
    {
        _productService = productService;
        _cache = cache;

        LoadCommand =
            new RelayCommand(
                async () =>
                    await LoadProductAsync());
    }

    private async Task LoadProductAsync()
    {
        if (_cache.TryGet(
            ProductId,
            out var cachedProduct))
        {
            Product = cachedProduct;
            Source = "Loaded from CACHE";
            return;
        }

        Product =
            await _productService.GetByIdAsync(
                ProductId);

        if (Product is not null)
            _cache.Set(Product);

        Source = "Loaded from SERVICE";
    }
}

ViewModels/ExportViewModel.cs

using System.Windows.Input;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Services;

namespace PerformanceTacticsDemo.ViewModels;

public class ExportViewModel : ObservableObject
{
    private readonly ProductService _productService;
    private readonly ExportService _exportService;

    private string _status = "Ready";
    private int _processed;

    public string Status
    {
        get => _status;
        set => SetProperty(
            ref _status,
            value);
    }

    public int Processed
    {
        get => _processed;
        set => SetProperty(
            ref _processed,
            value);
    }

    public ICommand ExportCommand { get; }

    public ExportViewModel(
        ProductService productService,
        ExportService exportService)
    {
        _productService = productService;
        _exportService = exportService;

        ExportCommand =
            new RelayCommand(
                async () => await ExportAsync());
    }

    private async Task ExportAsync()
    {
        Status = "Exporting...";
        Processed = 0;

        var progress =
            new Progress<int>(
                value => Processed = value);

        try
        {
            await _exportService.ExportAsync(
                _productService.StreamAllAsync(),
                chunkSize: 1000,
                maxConcurrency: 5,
                progress: progress);

            Status = "Completed";
        }
        catch (OperationCanceledException)
        {
            Status = "Cancelled";
        }
    }
}

ViewModels/LiveUpdatesViewModel.cs

using PerformanceTacticsDemo.Infrastructure;

namespace PerformanceTacticsDemo.ViewModels;

public class LiveUpdatesViewModel : ObservableObject
{
    private decimal _latestPrice;
    private decimal _displayedPrice;

    private readonly PeriodicTimer _timer =
        new(TimeSpan.FromMilliseconds(500));

    public decimal DisplayedPrice
    {
        get => _displayedPrice;
        set => SetProperty(
            ref _displayedPrice,
            value);
    }

    public LiveUpdatesViewModel()
    {
        _ = StartThrottledUpdatesAsync();
        _ = StartSimulatedProducerAsync();
    }

    private async Task StartThrottledUpdatesAsync()
    {
        while (await _timer.WaitForNextTickAsync())
            DisplayedPrice = _latestPrice;
    }

    private async Task StartSimulatedProducerAsync()
    {
        while (true)
        {
            _latestPrice =
                Random.Shared.Next(10, 1000);

            await Task.Delay(20);
        }
    }
}

ViewModels/MainViewModel.cs

using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Services;

namespace PerformanceTacticsDemo.ViewModels;

public class MainViewModel : ObservableObject
{
    public SearchViewModel Search { get; }
    public LiveUpdatesViewModel LiveUpdates { get; }
    public BatchUpdatesViewModel BatchUpdates { get; }
    public PriorityAlertsViewModel PriorityAlerts { get; }
    public PagingViewModel Paging { get; }
    public CacheViewModel Cache { get; }
    public TimeoutViewModel Timeout { get; }
    public ExportViewModel Export { get; }

    public MainViewModel(
        ProductService productService,
        ProductCache productCache,
        ExportService exportService)
    {
        Search = new SearchViewModel(productService);
        LiveUpdates = new LiveUpdatesViewModel();
        BatchUpdates = new BatchUpdatesViewModel();
        PriorityAlerts = new PriorityAlertsViewModel();
        Paging = new PagingViewModel(productService);
        Cache = new CacheViewModel(
            productService,
            productCache);
        Timeout = new TimeoutViewModel();
        Export = new ExportViewModel(
            productService,
            exportService);
    }
}

ViewModels/PagingViewModel.cs

using System.Collections.ObjectModel;
using System.Windows.Input;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Models;
using PerformanceTacticsDemo.Services;

namespace PerformanceTacticsDemo.ViewModels;

public class PagingViewModel : ObservableObject
{
    private readonly ProductService _productService;
    private int _currentPage = 1;

    public int PageSize { get; } = 100;

    public int CurrentPage
    {
        get => _currentPage;
        set => SetProperty(
            ref _currentPage,
            value);
    }

    public ObservableCollection<Product>
        Products { get; } = new();

    public ICommand NextPageCommand { get; }
    public ICommand PreviousPageCommand { get; }

    public PagingViewModel(
        ProductService productService)
    {
        _productService = productService;

        NextPageCommand = new RelayCommand(
            async () =>
            {
                CurrentPage++;
                await LoadPageAsync();
            });

        PreviousPageCommand = new RelayCommand(
            async () =>
            {
                if (CurrentPage > 1)
                    CurrentPage--;

                await LoadPageAsync();
            },
            () => CurrentPage > 1);

        _ = LoadPageAsync();
    }

    private async Task LoadPageAsync()
    {
        var page =
            await _productService.GetPageAsync(
                CurrentPage,
                PageSize);

        Products.Clear();

        foreach (var product in page)
            Products.Add(product);

        if (PreviousPageCommand
            is RelayCommand previous)
        {
            previous.RaiseCanExecuteChanged();
        }
    }
}

ViewModels/PriorityAlertsViewModel.cs

using System.Collections.ObjectModel;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Models;

namespace PerformanceTacticsDemo.ViewModels;

public class PriorityAlertsViewModel : ObservableObject
{
    private readonly PriorityWorkQueue<Alert> _queue = new();

    public ObservableCollection<Alert>
        ProcessedAlerts { get; } = new();

    public PriorityAlertsViewModel()
    {
        _queue.Enqueue(
            new Alert
            {
                Message = "Telemetry Update",
                Priority = AlertPriority.Low
            },
            (int)AlertPriority.Low);

        _queue.Enqueue(
            new Alert
            {
                Message = "User Order Failed",
                Priority = AlertPriority.High
            },
            (int)AlertPriority.High);

        _queue.Enqueue(
            new Alert
            {
                Message = "System Down",
                Priority = AlertPriority.Critical
            },
            (int)AlertPriority.Critical);
    }

    public void ProcessNext()
    {
        if (_queue.TryDequeue(out var alert) &&
            alert is not null)
        {
            ProcessedAlerts.Add(alert);
        }
    }
}

ViewModels/SearchViewModel.cs

using System.Collections.ObjectModel;
using PerformanceTacticsDemo.Infrastructure;
using PerformanceTacticsDemo.Models;
using PerformanceTacticsDemo.Services;

namespace PerformanceTacticsDemo.ViewModels;

public class SearchViewModel : ObservableObject
{
    private readonly ProductService _productService;
    private readonly Debouncer _debouncer = new();
    private string _searchText = string.Empty;

    public ObservableCollection<Product> Results { get; } = new();

    public string SearchText
    {
        get => _searchText;
        set
        {
            if (SetProperty(
                ref _searchText,
                value))
            {
                _ = SearchDebouncedAsync(value);
            }
        }
    }

    public SearchViewModel(
        ProductService productService)
    {
        _productService = productService;
    }

    private async Task SearchDebouncedAsync(
        string searchText)
    {
        await _debouncer.ExecuteAsync(
            TimeSpan.FromMilliseconds(500),
            async cancellationToken =>
            {
                var products =
                    await _productService.SearchAsync(
                        searchText,
                        cancellationToken);

                Results.Clear();

                foreach (var product in products)
                    Results.Add(product);
            });
    }
}

ViewModels/TimeoutViewModel.cs

using System.Windows.Input;
using PerformanceTacticsDemo.Infrastructure;

namespace PerformanceTacticsDemo.ViewModels;

public class TimeoutViewModel : ObservableObject
{
    private CancellationTokenSource? _cts;
    private string _status = "Ready";

    public string Status
    {
        get => _status;
        set => SetProperty(
            ref _status,
            value);
    }

    public ICommand StartCommand { get; }
    public ICommand CancelCommand { get; }

    public TimeoutViewModel()
    {
        StartCommand =
            new RelayCommand(
                async () => await StartAsync());

        CancelCommand =
            new RelayCommand(
                () => _cts?.Cancel());
    }

    private async Task StartAsync()
    {
        _cts?.Cancel();
        _cts = new CancellationTokenSource();

        using var timeoutCts =
            new CancellationTokenSource(
                TimeSpan.FromSeconds(5));

        using var linkedCts =
            CancellationTokenSource
                .CreateLinkedTokenSource(
                    _cts.Token,
                    timeoutCts.Token);

        try
        {
            Status = "Processing...";

            await SimulateSlowOperationAsync(
                linkedCts.Token);

            Status = "Completed";
        }
        catch (OperationCanceledException)
        {
            Status =
                timeoutCts.IsCancellationRequested
                    ? "Timed out"
                    : "Cancelled";
        }
    }

    private static Task
        SimulateSlowOperationAsync(
            CancellationToken token) =>
        Task.Delay(
            TimeSpan.FromSeconds(10),
            token);
}

Views/BatchUpdatesView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.BatchUpdatesView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding BatchUpdates}">
        <TextBlock Text="Batching + Event Coalescing"
                   FontSize="20"
                   FontWeight="Bold"/>
        <Button Content="Generate Demo Updates"
                Click="GenerateUpdates_Click"
                Margin="0,10"/>
        <ListBox Height="350"
                 ItemsSource="{Binding ProcessedUpdates}"/>
    </StackPanel>
</UserControl>

Views/BatchUpdatesView.xaml.cs

using System.Windows;
using System.Windows.Controls;
using PerformanceTacticsDemo.Models;
using PerformanceTacticsDemo.ViewModels;

namespace PerformanceTacticsDemo.Views;

public partial class BatchUpdatesView : UserControl
{
    public BatchUpdatesView() => InitializeComponent();

    private void GenerateUpdates_Click(object sender, RoutedEventArgs e)
    {
        if (DataContext is BatchUpdatesViewModel vm)
        {
            for (int i = 0; i < 20; i++)
            {
                vm.ReceiveUpdate(new ProductUpdate
                {
                    ProductId = (i % 3) + 1,
                    NewPrice = 100 + i,
                    Timestamp = DateTime.Now
                });
            }
        }
    }
}

Views/CacheView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.CacheView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding Cache}">
        <TextBlock Text="Product Details - Caching"
                   FontSize="20"
                   FontWeight="Bold"/>
        <TextBox Text="{Binding ProductId}"
                 Margin="0,10"/>
        <Button Content="Load Product"
                Command="{Binding LoadCommand}"
                Margin="0,5"/>
        <TextBlock Text="{Binding Product}"
                   Margin="0,10"/>
        <TextBlock Text="{Binding Source}"/>
    </StackPanel>
</UserControl>

Views/CacheView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class CacheView : UserControl
{
    public CacheView() => InitializeComponent();
}

Views/ExportView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.ExportView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding Export}">
        <TextBlock Text="Export - Streaming + Chunking + Concurrency Control"
                   FontSize="20"
                   FontWeight="Bold"
                   TextWrapping="Wrap"/>
        <Button Content="Start Export"
                Command="{Binding ExportCommand}"
                Margin="0,15"/>
        <TextBlock Text="{Binding Status}"/>
        <TextBlock Text="{Binding Processed}"/>
    </StackPanel>
</UserControl>

Views/ExportView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class ExportView : UserControl
{
    public ExportView() => InitializeComponent();
}

Views/LiveUpdatesView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.LiveUpdatesView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding LiveUpdates}">
        <TextBlock Text="Live Updates - Latest-Value Throttling"
                   FontSize="20"
                   FontWeight="Bold"/>
        <TextBlock Text="{Binding DisplayedPrice}"
                   FontSize="40"
                   Margin="0,20"/>
    </StackPanel>
</UserControl>

Views/LiveUpdatesView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class LiveUpdatesView : UserControl
{
    public LiveUpdatesView() => InitializeComponent();
}

Views/PagingView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.PagingView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Grid Margin="20"
          DataContext="{Binding Paging}">
        <Grid.RowDefinitions>
            <RowDefinition/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <DataGrid ItemsSource="{Binding Products}"
                  EnableRowVirtualization="True"
                  EnableColumnVirtualization="True"
                  VirtualizingPanel.IsVirtualizing="True"
                  VirtualizingPanel.VirtualizationMode="Recycling"/>

        <StackPanel Grid.Row="1"
                    Orientation="Horizontal"
                    HorizontalAlignment="Center">
            <Button Content="Previous"
                    Command="{Binding PreviousPageCommand}"
                    Margin="5"/>
            <TextBlock Text="{Binding CurrentPage}"
                       VerticalAlignment="Center"
                       Margin="10"/>
            <Button Content="Next"
                    Command="{Binding NextPageCommand}"
                    Margin="5"/>
        </StackPanel>
    </Grid>
</UserControl>

Views/PagingView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class PagingView : UserControl
{
    public PagingView() => InitializeComponent();
}

Views/PriorityAlertsView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.PriorityAlertsView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding PriorityAlerts}">
        <TextBlock Text="Priority Queue"
                   FontSize="20"
                   FontWeight="Bold"/>
        <Button Content="Process Next Alert"
                Click="ProcessNext_Click"
                Margin="0,10"/>
        <ListBox Height="300"
                 ItemsSource="{Binding ProcessedAlerts}"/>
    </StackPanel>
</UserControl>

Views/PriorityAlertsView.xaml.cs

using System.Windows;
using System.Windows.Controls;
using PerformanceTacticsDemo.ViewModels;

namespace PerformanceTacticsDemo.Views;

public partial class PriorityAlertsView : UserControl
{
    public PriorityAlertsView() => InitializeComponent();

    private void ProcessNext_Click(object sender, RoutedEventArgs e)
    {
        if (DataContext is PriorityAlertsViewModel vm)
            vm.ProcessNext();
    }
}

Views/SearchView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.SearchView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding Search}">
        <TextBlock Text="Search - Debouncing"
                   FontSize="20"
                   FontWeight="Bold"/>
        <TextBox Margin="0,10"
                 Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"/>
        <ListBox Height="350"
                 ItemsSource="{Binding Results}"/>
    </StackPanel>
</UserControl>

Views/SearchView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class SearchView : UserControl
{
    public SearchView() => InitializeComponent();
}

Views/TimeoutView.xaml

<UserControl x:Class="PerformanceTacticsDemo.Views.TimeoutView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Margin="20"
                DataContext="{Binding Timeout}">
        <TextBlock Text="Timeout + Cancellation"
                   FontSize="20"
                   FontWeight="Bold"/>
        <StackPanel Orientation="Horizontal"
                    Margin="0,15">
            <Button Content="Start"
                    Command="{Binding StartCommand}"
                    Margin="5"/>
            <Button Content="Cancel"
                    Command="{Binding CancelCommand}"
                    Margin="5"/>
        </StackPanel>
        <TextBlock Text="{Binding Status}"
                   FontSize="18"/>
    </StackPanel>
</UserControl>

Views/TimeoutView.xaml.cs

using System.Windows.Controls;

namespace PerformanceTacticsDemo.Views;

public partial class TimeoutView : UserControl
{
    public TimeoutView() => InitializeComponent();
}
