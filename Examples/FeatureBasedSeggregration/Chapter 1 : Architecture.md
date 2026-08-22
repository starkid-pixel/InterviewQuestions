Yes. Below is a complete starting architecture for the segregated 5-module WPF application.

The key change from the previous version is:

OLD
MainWindow
 └── TabControl
      ├── SearchView
      ├── PagingView
      ├── CacheView
      └── etc.

NEW
Shell
 ├── Navigation Menu
 └── Content Area
       │
       └── Active Module
            ├── ProductSearch
            ├── LiveMonitoring
            ├── DataManagement
            ├── Processing
            └── Reporting

We will use:

WPF
MVVM
Dependency Injection
Shell
Navigation Service
Feature-based modules
Caching
Debouncing
Throttling
Paging
Batching
Priority processing
Timeout/Cancellation
Streaming/Chunking
1. Project structure
PerformanceArchitectureApp
│
├── PerformanceArchitectureApp.csproj
│
├── App.xaml
├── App.xaml.cs
│
├── Shell
│   ├── ShellView.xaml
│   ├── ShellView.xaml.cs
│   └── ShellViewModel.cs
│
├── Navigation
│   ├── INavigationService.cs
│   ├── NavigationService.cs
│   └── NavigationItem.cs
│
├── Shared
│   │
│   ├── Infrastructure
│   │   ├── ObservableObject.cs
│   │   └── RelayCommand.cs
│   │
│   └── Models
│       ├── Product.cs
│       ├── Alert.cs
│       └── WorkItem.cs
│
├── Modules
│   │
│   ├── ProductSearch
│   │   ├── ProductSearchView.xaml
│   │   ├── ProductSearchView.xaml.cs
│   │   ├── ProductSearchViewModel.cs
│   │   ├── ProductSearchService.cs
│   │   └── Debouncer.cs
│   │
│   ├── LiveMonitoring
│   │   ├── LiveMonitoringView.xaml
│   │   ├── LiveMonitoringView.xaml.cs
│   │   ├── LiveMonitoringViewModel.cs
│   │   └── LiveDataService.cs
│   │
│   ├── DataManagement
│   │   ├── DataManagementView.xaml
│   │   ├── DataManagementView.xaml.cs
│   │   ├── DataManagementViewModel.cs
│   │   ├── ProductRepository.cs
│   │   └── ProductCache.cs
│   │
│   ├── Processing
│   │   ├── ProcessingView.xaml
│   │   ├── ProcessingView.xaml.cs
│   │   ├── ProcessingViewModel.cs
│   │   └── BatchProcessor.cs
│   │
│   └── Reporting
│       ├── ReportingView.xaml
│       ├── ReportingView.xaml.cs
│       ├── ReportingViewModel.cs
│       └── ReportExportService.cs
2. Project file
PerformanceArchitectureApp.csproj
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>

        <OutputType>WinExe</OutputType>

        <TargetFramework>net8.0-windows</TargetFramework>

        <UseWPF>true</UseWPF>

        <Nullable>enable</Nullable>

        <ImplicitUsings>enable</ImplicitUsings>

    </PropertyGroup>

    <ItemGroup>

        <PackageReference
            Include="Microsoft.Extensions.DependencyInjection"
            Version="8.0.1" />

    </ItemGroup>

</Project>
3. Application startup
App.xaml
<Application
    x:Class="PerformanceArchitectureApp.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Application.Resources>

    </Application.Resources>

</Application>

Notice:

No StartupUri

Because the Composition Root controls startup.

4. App.xaml.cs
using Microsoft.Extensions.DependencyInjection;

using PerformanceArchitectureApp.Navigation;

using PerformanceArchitectureApp.Shell;

using PerformanceArchitectureApp.Modules.ProductSearch;

using PerformanceArchitectureApp.Modules.LiveMonitoring;

using PerformanceArchitectureApp.Modules.DataManagement;

using PerformanceArchitectureApp.Modules.Processing;

using PerformanceArchitectureApp.Modules.Reporting;

using System.Windows;


namespace PerformanceArchitectureApp;

public partial class App : Application
{
    private ServiceProvider? _serviceProvider;


    protected override void OnStartup(
        StartupEventArgs e)
    {
        base.OnStartup(e);


        var services =
            new ServiceCollection();


        ConfigureServices(services);


        _serviceProvider =
            services.BuildServiceProvider();


        var shell =
            _serviceProvider
                .GetRequiredService<ShellView>();


        shell.Show();
    }


    private static void ConfigureServices(
        IServiceCollection services)
    {
        /*
         * ==========================
         * Navigation
         * ==========================
         */

        services.AddSingleton<INavigationService>(
            NavigationServiceFactory);


        /*
         * ==========================
         * Module 1
         * Product Search
         * ==========================
         */

        services.AddSingleton<ProductSearchService>();

        services.AddSingleton<ProductSearchViewModel>();

        services.AddSingleton<ProductSearchView>();


        /*
         * ==========================
         * Module 2
         * Live Monitoring
         * ==========================
         */

        services.AddSingleton<LiveDataService>();

        services.AddSingleton<LiveMonitoringViewModel>();

        services.AddSingleton<LiveMonitoringView>();


        /*
         * ==========================
         * Module 3
         * Data Management
         * ==========================
         */

        services.AddSingleton<ProductRepository>();

        services.AddSingleton<ProductCache>();

        services.AddSingleton<DataManagementViewModel>();

        services.AddSingleton<DataManagementView>();


        /*
         * ==========================
         * Module 4
         * Processing
         * ==========================
         */

        services.AddSingleton<BatchProcessor>();

        services.AddSingleton<ProcessingViewModel>();

        services.AddSingleton<ProcessingView>();


        /*
         * ==========================
         * Module 5
         * Reporting
         * ==========================
         */

        services.AddSingleton<ReportExportService>();

        services.AddSingleton<ReportingViewModel>();

        services.AddSingleton<ReportingView>();


        /*
         * ==========================
         * Shell
         * ==========================
         */

        services.AddSingleton<ShellViewModel>();

        services.AddSingleton<ShellView>();
    }


    private static INavigationService
        NavigationServiceFactory(
            IServiceProvider provider)
    {
        return new NavigationService(
            provider);
    }


    protected override void OnExit(
        ExitEventArgs e)
    {
        _serviceProvider?.Dispose();

        base.OnExit(e);
    }
}

This is the Composition Root.

It knows:

What objects exist?
How are they created?
What depends on what?

But it does not contain business logic.

5. Shared infrastructure
Shared/Infrastructure/ObservableObject.cs
using System.ComponentModel;

using System.Runtime.CompilerServices;


namespace PerformanceArchitectureApp.Shared.Infrastructure;

public abstract class ObservableObject
    : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler?
        PropertyChanged;


    protected void OnPropertyChanged(
        [CallerMemberName]
        string? propertyName = null)
    {
        PropertyChanged?.Invoke(
            this,

            new PropertyChangedEventArgs(
                propertyName));
    }


    protected bool SetProperty<T>(
        ref T field,

        T value,

        [CallerMemberName]
        string? propertyName = null)
    {
        if (EqualityComparer<T>
            .Default
            .Equals(field, value))
        {
            return false;
        }


        field = value;


        OnPropertyChanged(
            propertyName);


        return true;
    }
}
Shared/Infrastructure/RelayCommand.cs
using System.Windows.Input;


namespace PerformanceArchitectureApp.Shared.Infrastructure;

public sealed class RelayCommand
    : ICommand
{
    private readonly Action _execute;

    private readonly Func<bool>?
        _canExecute;


    public RelayCommand(
        Action execute,

        Func<bool>? canExecute = null)
    {
        _execute = execute;

        _canExecute = canExecute;
    }


    public event EventHandler?
        CanExecuteChanged;


    public bool CanExecute(
        object? parameter)
    {
        return _canExecute?.Invoke()
            ?? true;
    }


    public void Execute(
        object? parameter)
    {
        _execute();
    }


    public void RaiseCanExecuteChanged()
    {
        CanExecuteChanged?.Invoke(
            this,

            EventArgs.Empty);
    }
}
6. Shared models
Product.cs
namespace PerformanceArchitectureApp.Shared.Models;

public sealed class Product
{
    public int Id { get; init; }

    public string Name { get; init; }
        = string.Empty;

    public decimal Price { get; init; }

    public string Category { get; init; }
        = string.Empty;
}
Alert.cs
namespace PerformanceArchitectureApp.Shared.Models;

public sealed class Alert
{
    public string Message { get; init; }
        = string.Empty;

    public int Priority { get; init; }
}
7. Navigation

This is the important new part.

Navigation/INavigationService.cs
using PerformanceArchitectureApp.Shared.Infrastructure;


namespace PerformanceArchitectureApp.Navigation;

public interface INavigationService
{
    ObservableObject?
        CurrentViewModel
    {
        get;
    }


    void NavigateTo<TViewModel>()
        where TViewModel
            : ObservableObject;
}
Navigation/NavigationService.cs
using Microsoft.Extensions.DependencyInjection;

using PerformanceArchitectureApp.Shared.Infrastructure;


namespace PerformanceArchitectureApp.Navigation;

public sealed class NavigationService
    : ObservableObject,
      INavigationService
{
    private readonly IServiceProvider
        _serviceProvider;


    private ObservableObject?
        _currentViewModel;


    public ObservableObject?
        CurrentViewModel
    {
        get => _currentViewModel;

        private set =>
            SetProperty(
                ref _currentViewModel,
                value);
    }


    public NavigationService(
        IServiceProvider serviceProvider)
    {
        _serviceProvider =
            serviceProvider;
    }


    public void NavigateTo<TViewModel>()
        where TViewModel
            : ObservableObject
    {
        CurrentViewModel =
            _serviceProvider
                .GetRequiredService<TViewModel>();
    }
}

This is the router.

User clicks menu
       ↓
ShellViewModel
       ↓
NavigationService
       ↓
Resolve ViewModel
       ↓
CurrentViewModel changes
       ↓
Content area changes
8. Shell
ShellViewModel.cs
using PerformanceArchitectureApp.Navigation;

using PerformanceArchitectureApp.Shared.Infrastructure;

using PerformanceArchitectureApp.Modules.ProductSearch;

using PerformanceArchitectureApp.Modules.LiveMonitoring;

using PerformanceArchitectureApp.Modules.DataManagement;

using PerformanceArchitectureApp.Modules.Processing;

using PerformanceArchitectureApp.Modules.Reporting;


namespace PerformanceArchitectureApp.Shell;

public sealed class ShellViewModel
    : ObservableObject
{
    private readonly INavigationService
        _navigationService;


    public ObservableObject?
        CurrentViewModel
        =>
        _navigationService
            .CurrentViewModel;


    public RelayCommand
        NavigateToSearchCommand { get; }


    public RelayCommand
        NavigateToMonitoringCommand { get; }


    public RelayCommand
        NavigateToDataCommand { get; }


    public RelayCommand
        NavigateToProcessingCommand { get; }


    public RelayCommand
        NavigateToReportingCommand { get; }


    public ShellViewModel(
        INavigationService navigationService)
    {
        _navigationService =
            navigationService;


        NavigateToSearchCommand =
            new RelayCommand(
                NavigateToSearch);


        NavigateToMonitoringCommand =
            new RelayCommand(
                NavigateToMonitoring);


        NavigateToDataCommand =
            new RelayCommand(
                NavigateToData);


        NavigateToProcessingCommand =
            new RelayCommand(
                NavigateToProcessing);


        NavigateToReportingCommand =
            new RelayCommand(
                NavigateToReporting);


        NavigateToSearch();
    }


    private void NavigateToSearch()
    {
        _navigationService
            .NavigateTo<ProductSearchViewModel>();

        OnPropertyChanged(
            nameof(CurrentViewModel));
    }


    private void NavigateToMonitoring()
    {
        _navigationService
            .NavigateTo<LiveMonitoringViewModel>();

        OnPropertyChanged(
            nameof(CurrentViewModel));
    }


    private void NavigateToData()
    {
        _navigationService
            .NavigateTo<DataManagementViewModel>();

        OnPropertyChanged(
            nameof(CurrentViewModel));
    }


    private void NavigateToProcessing()
    {
        _navigationService
            .NavigateTo<ProcessingViewModel>();

        OnPropertyChanged(
            nameof(CurrentViewModel));
    }


    private void NavigateToReporting()
    {
        _navigationService
            .NavigateTo<ReportingViewModel>();

        OnPropertyChanged(
            nameof(CurrentViewModel));
    }
}
ShellView.xaml
<Window
    x:Class="
        PerformanceArchitectureApp.Shell.ShellView"

    xmlns="
        http://schemas.microsoft.com/winfx/2006/xaml/presentation"

    xmlns:x="
        http://schemas.microsoft.com/winfx/2006/xaml"

    xmlns:search="
        clr-namespace:
        PerformanceArchitectureApp.Modules.ProductSearch"

    xmlns:monitoring="
        clr-namespace:
        PerformanceArchitectureApp.Modules.LiveMonitoring"

    xmlns:data="
        clr-namespace:
        PerformanceArchitectureApp.Modules.DataManagement"

    xmlns:processing="
        clr-namespace:
        PerformanceArchitectureApp.Modules.Processing"

    xmlns:reporting="
        clr-namespace:
        PerformanceArchitectureApp.Modules.Reporting"

    Title="Performance Architecture Application"

    Height="700"

    Width="1200">


    <Window.Resources>


        <DataTemplate
            DataType=
            "{x:Type search:ProductSearchViewModel}">

            <search:ProductSearchView />

        </DataTemplate>


        <DataTemplate
            DataType=
            "{x:Type monitoring:LiveMonitoringViewModel}">

            <monitoring:LiveMonitoringView />

        </DataTemplate>


        <DataTemplate
            DataType=
            "{x:Type data:DataManagementViewModel}">

            <data:DataManagementView />

        </DataTemplate>


        <DataTemplate
            DataType=
            "{x:Type processing:ProcessingViewModel}">

            <processing:ProcessingView />

        </DataTemplate>


        <DataTemplate
            DataType=
            "{x:Type reporting:ReportingViewModel}">

            <reporting:ReportingView />

        </DataTemplate>


    </Window.Resources>


    <Grid>


        <Grid.ColumnDefinitions>

            <ColumnDefinition
                Width="220" />

            <ColumnDefinition
                Width="*" />

        </Grid.ColumnDefinitions>


        <!-- Navigation -->

        <StackPanel
            Background="LightGray">


            <TextBlock

                Text="MODULES"

                Margin="20"

                FontSize="20"

                FontWeight="Bold" />


            <Button

                Content="1. Product Search"

                Margin="10"

                Command=
                "{Binding NavigateToSearchCommand}" />


            <Button

                Content="2. Live Monitoring"

                Margin="10"

                Command=
                "{Binding NavigateToMonitoringCommand}" />


            <Button

                Content="3. Data Management"

                Margin="10"

                Command=
                "{Binding NavigateToDataCommand}" />


            <Button

                Content="4. Processing"

                Margin="10"

                Command=
                "{Binding NavigateToProcessingCommand}" />


            <Button

                Content="5. Reporting"

                Margin="10"

                Command=
                "{Binding NavigateToReportingCommand}" />


        </StackPanel>


        <!-- Content Region -->

        <ContentControl

            Grid.Column="1"

            Content=
            "{Binding CurrentViewModel}" />


    </Grid>

</Window>

The important part:

<ContentControl
    Content="{Binding CurrentViewModel}" />

This is the equivalent of a Content Region.

The DataTemplate decides:

ProductSearchViewModel
        ↓
ProductSearchView

DataManagementViewModel
        ↓
DataManagementView
ShellView.xaml.cs
using System.Windows;


namespace PerformanceArchitectureApp.Shell;

public partial class ShellView
    : Window
{
    public ShellView(
        ShellViewModel viewModel)
    {
        InitializeComponent();

        DataContext =
            viewModel;
    }
}

This code-behind is only injecting the root ViewModel.

You could also set it in App.xaml.cs. Both approaches work. For consistency, I prefer constructor injection here because the View has exactly one required root ViewModel.

MODULE 1 — PRODUCT SEARCH
ProductSearchService.cs
using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.ProductSearch;

public sealed class ProductSearchService
{
    private readonly List<Product>
        _products;


    public ProductSearchService()
    {
        _products =
            Enumerable
                .Range(1, 100_000)

                .Select(i =>
                    new Product
                    {
                        Id = i,

                        Name =
                            $"Product {i}",

                        Price =
                            Random.Shared.Next(
                                10,
                                10_000),

                        Category =
                            $"Category {i % 10}"
                    })

                .ToList();
    }


    public async Task<List<Product>>
        SearchAsync(
            string text,

            CancellationToken token)
    {
        await Task.Delay(
            300,
            token);


        return _products

            .Where(p =>
                p.Name.Contains(
                    text,

                    StringComparison
                        .OrdinalIgnoreCase))

            .Take(100)

            .ToList();
    }
}
Debouncer.cs
namespace PerformanceArchitectureApp.Modules.ProductSearch;

public sealed class Debouncer
{
    private CancellationTokenSource?
        _cancellationTokenSource;


    public async Task ExecuteAsync(
        TimeSpan delay,

        Func<CancellationToken, Task>
            action)
    {
        var newTokenSource =
            new CancellationTokenSource();


        var oldTokenSource =
            Interlocked.Exchange(

                ref _cancellationTokenSource,

                newTokenSource);


        oldTokenSource?.Cancel();

        oldTokenSource?.Dispose();


        try
        {
            await Task.Delay(
                delay,

                newTokenSource.Token);


            await action(
                newTokenSource.Token);
        }

        catch (
            OperationCanceledException)
        {
            // New input replaced old input.
        }
    }
}
ProductSearchViewModel.cs
using System.Collections.ObjectModel;

using PerformanceArchitectureApp.Shared.Infrastructure;

using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.ProductSearch;

public sealed class ProductSearchViewModel
    : ObservableObject
{
    private readonly ProductSearchService
        _service;


    private readonly Debouncer
        _debouncer = new();


    private string _searchText =
        string.Empty;


    public ObservableCollection<Product>
        Results { get; }
        = new();


    public string SearchText
    {
        get => _searchText;

        set
        {
            if (SetProperty(
                ref _searchText,
                value))
            {
                _ = SearchAsync(value);
            }
        }
    }


    public ProductSearchViewModel(
        ProductSearchService service)
    {
        _service =
            service;
    }


    private async Task SearchAsync(
        string text)
    {
        await _debouncer.ExecuteAsync(

            TimeSpan.FromMilliseconds(500),

            async token =>
            {
                var results =
                    await _service
                        .SearchAsync(
                            text,
                            token);


                Results.Clear();


                foreach (
                    var product in results)
                {
                    Results.Add(product);
                }
            });
    }
}
ProductSearchView.xaml
<UserControl
    x:Class="
    PerformanceArchitectureApp.Modules.ProductSearch.ProductSearchView"

    xmlns="
    http://schemas.microsoft.com/winfx/2006/xaml/presentation"

    xmlns:x="
    http://schemas.microsoft.com/winfx/2006/xaml">


    <Grid Margin="20">


        <Grid.RowDefinitions>

            <RowDefinition
                Height="Auto" />

            <RowDefinition
                Height="Auto" />

            <RowDefinition />

        </Grid.RowDefinitions>


        <TextBlock

            Text="Product Search - Debouncing"

            FontSize="24"

            FontWeight="Bold" />


        <TextBox

            Grid.Row="1"

            Margin="0,20"

            Text=
            "{Binding SearchText,
              UpdateSourceTrigger=PropertyChanged}" />


        <DataGrid

            Grid.Row="2"

            ItemsSource="{Binding Results}"

            IsReadOnly="True"

            EnableRowVirtualization="True"

            EnableColumnVirtualization="True" />


    </Grid>

</UserControl>
MODULE 2 — LIVE MONITORING
LiveDataService.cs
namespace PerformanceArchitectureApp.Modules.LiveMonitoring;

public sealed class LiveDataService
{
    public async IAsyncEnumerable<decimal>
        StreamAsync(
            [System.Runtime.CompilerServices
                .EnumeratorCancellation]
            CancellationToken token)
    {
        while (
            !token.IsCancellationRequested)
        {
            yield return
                Random.Shared.Next(
                    10,
                    1000);


            await Task.Delay(
                20,
                token);
        }
    }
}
LiveMonitoringViewModel.cs
using PerformanceArchitectureApp.Shared.Infrastructure;


namespace PerformanceArchitectureApp.Modules.LiveMonitoring;

public sealed class LiveMonitoringViewModel
    : ObservableObject
{
    private readonly LiveDataService
        _service;


    private decimal _latestValue;

    private decimal _displayedValue;

    private int _receivedEvents;

    private int _uiUpdates;


    public decimal DisplayedValue
    {
        get => _displayedValue;

        private set =>
            SetProperty(
                ref _displayedValue,
                value);
    }


    public int ReceivedEvents
    {
        get => _receivedEvents;

        private set =>
            SetProperty(
                ref _receivedEvents,
                value);
    }


    public int UiUpdates
    {
        get => _uiUpdates;

        private set =>
            SetProperty(
                ref _uiUpdates,
                value);
    }


    public LiveMonitoringViewModel(
        LiveDataService service)
    {
        _service = service;

        _ = StartAsync();
    }


    private async Task StartAsync()
    {
        using var cts =
            new CancellationTokenSource();


        _ =
            Task.Run(
                async () =>
                {
                    await foreach (
                        var value
                        in _service.StreamAsync(
                            cts.Token))
                    {
                        _latestValue =
                            value;

                        ReceivedEvents++;
                    }
                });


        using var timer =
            new PeriodicTimer(

                TimeSpan
                    .FromMilliseconds(500));


        while (
            await timer
                .WaitForNextTickAsync())
        {
            DisplayedValue =
                _latestValue;

            UiUpdates++;
        }
    }
}

This demonstrates:

Incoming:
50 events / second

UI:
2 updates / second

That is throttling / sampling.

MODULE 3 — DATA MANAGEMENT
ProductRepository.cs
using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.DataManagement;

public sealed class ProductRepository
{
    private readonly List<Product>
        _products;


    public ProductRepository()
    {
        _products =
            Enumerable
                .Range(1, 100_000)

                .Select(i =>
                    new Product
                    {
                        Id = i,

                        Name =
                            $"Product {i}",

                        Price =
                            Random.Shared.Next(
                                10,
                                5000),

                        Category =
                            $"Category {i % 5}"
                    })

                .ToList();
    }


    public async Task<List<Product>>
        GetPageAsync(
            int page,

            int pageSize)
    {
        await Task.Delay(100);


        return _products

            .Skip(
                (page - 1)
                * pageSize)

            .Take(pageSize)

            .ToList();
    }


    public async Task<Product?>
        GetByIdAsync(
            int id)
    {
        await Task.Delay(300);


        return _products
            .FirstOrDefault(
                x => x.Id == id);
    }
}
ProductCache.cs
using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.DataManagement;

public sealed class ProductCache
{
    private readonly Dictionary<int, Product>
        _cache = new();


    public bool TryGet(
        int id,

        out Product? product)
    {
        return _cache.TryGetValue(
            id,

            out product);
    }


    public void Set(
        Product product)
    {
        _cache[product.Id] =
            product;
    }


    public void Clear()
    {
        _cache.Clear();
    }
}
DataManagementViewModel.cs
using System.Collections.ObjectModel;

using PerformanceArchitectureApp.Shared.Infrastructure;

using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.DataManagement;

public sealed class DataManagementViewModel
    : ObservableObject
{
    private readonly ProductRepository
        _repository;

    private readonly ProductCache
        _cache;


    private int _page = 1;

    private string _status =
        "Ready";


    public ObservableCollection<Product>
        Products { get; }
        = new();


    public int Page
    {
        get => _page;

        private set =>
            SetProperty(
                ref _page,
                value);
    }


    public string Status
    {
        get => _status;

        private set =>
            SetProperty(
                ref _status,
                value);
    }


    public RelayCommand
        NextPageCommand { get; }


    public RelayCommand
        PreviousPageCommand { get; }


    public DataManagementViewModel(
        ProductRepository repository,

        ProductCache cache)
    {
        _repository = repository;

        _cache = cache;


        NextPageCommand =
            new RelayCommand(
                () => _ = NextPageAsync());


        PreviousPageCommand =
            new RelayCommand(
                () => _ = PreviousPageAsync(),

                () => Page > 1);


        _ = LoadPageAsync();
    }


    private async Task NextPageAsync()
    {
        Page++;

        PreviousPageCommand
            .RaiseCanExecuteChanged();

        await LoadPageAsync();
    }


    private async Task PreviousPageAsync()
    {
        if (Page <= 1)
        {
            return;
        }


        Page--;

        PreviousPageCommand
            .RaiseCanExecuteChanged();

        await LoadPageAsync();
    }


    private async Task LoadPageAsync()
    {
        Status =
            "Loading page...";


        var products =
            await _repository
                .GetPageAsync(
                    Page,
                    100);


        Products.Clear();


        foreach (
            var product in products)
        {
            Products.Add(product);

            _cache.Set(product);
        }


        Status =
            $"Page {Page} loaded";
    }
}

This module demonstrates:

Large Dataset
     │
     ├── Paging
     │
     └── Caching
MODULE 4 — PROCESSING
BatchProcessor.cs
namespace PerformanceArchitectureApp.Modules.Processing;

public sealed class BatchProcessor
{
    public async Task<int>
        ProcessAsync(
            IEnumerable<int> items,

            int batchSize,

            CancellationToken token)
    {
        var processed = 0;


        foreach (
            var batch
            in items.Chunk(batchSize))
        {
            token.ThrowIfCancellationRequested();


            await Task.Delay(
                200,
                token);


            processed +=
                batch.Length;
        }


        return processed;
    }
}
ProcessingViewModel.cs
using PerformanceArchitectureApp.Shared.Infrastructure;


namespace PerformanceArchitectureApp.Modules.Processing;

public sealed class ProcessingViewModel
    : ObservableObject
{
    private readonly BatchProcessor
        _processor;


    private string _status =
        "Ready";


    public string Status
    {
        get => _status;

        private set =>
            SetProperty(
                ref _status,
                value);
    }


    public RelayCommand
        StartCommand { get; }


    public ProcessingViewModel(
        BatchProcessor processor)
    {
        _processor = processor;


        StartCommand =
            new RelayCommand(
                () => _ = StartAsync());
    }


    private async Task StartAsync()
    {
        using var timeout =
            new CancellationTokenSource(

                TimeSpan.FromSeconds(5));


        try
        {
            Status =
                "Processing batches...";


            var result =
                await _processor
                    .ProcessAsync(

                        Enumerable.Range(
                            1,
                            10_000),

                        500,

                        timeout.Token);


            Status =
                $"Processed {result} items";
        }

        catch (
            OperationCanceledException)
        {
            Status =
                "Processing timed out";
        }
    }
}

This module demonstrates:

Work
 │
 ├── Batching
 │
 ├── Chunking
 │
 └── Bound Execution Time
MODULE 5 — REPORTING
ReportExportService.cs
using PerformanceArchitectureApp.Shared.Models;


namespace PerformanceArchitectureApp.Modules.Reporting;

public sealed class ReportExportService
{
    public async IAsyncEnumerable<Product>
        ExportAsync(
            [System.Runtime.CompilerServices
                .EnumeratorCancellation]
            CancellationToken token)
    {
        for (
            int i = 1;

            i <= 100_000;

            i++)
        {
            token.ThrowIfCancellationRequested();


            await Task.Delay(
                1,
                token);


            yield return
                new Product
                {
                    Id = i,

                    Name =
                        $"Product {i}",

                    Price =
                        Random.Shared.Next(
                            10,
                            1000),

                    Category =
                        "Export"
                };
        }
    }
}
ReportingViewModel.cs
using PerformanceArchitectureApp.Shared.Infrastructure;


namespace PerformanceArchitectureApp.Modules.Reporting;

public sealed class ReportingViewModel
    : ObservableObject
{
    private readonly ReportExportService
        _service;


    private int _processed;

    private string _status =
        "Ready";


    public int Processed
    {
        get => _processed;

        private set =>
            SetProperty(
                ref _processed,
                value);
    }


    public string Status
    {
        get => _status;

        private set =>
            SetProperty(
                ref _status,
                value);
    }


    public RelayCommand
        ExportCommand { get; }


    public ReportingViewModel(
        ReportExportService service)
    {
        _service = service;


        ExportCommand =
            new RelayCommand(
                () => _ = ExportAsync());
    }


    private async Task ExportAsync()
    {
        Processed = 0;

        Status =
            "Streaming data...";


        await foreach (
            var batch

            in _service
                .ExportAsync(
                    CancellationToken.None)

                .ChunkAsync(
                    500))
        {
            Processed +=
                batch.Length;


            Status =
                $"Processed {Processed} records";
        }


        Status =
            "Export complete";
    }
}

For ChunkAsync, add this extension.

AsyncEnumerableExtensions.cs
namespace PerformanceArchitectureApp.Shared.Infrastructure;

public static class AsyncEnumerableExtensions
{
    public static async IAsyncEnumerable<T[]>
        ChunkAsync<T>(
            this IAsyncEnumerable<T> source,

            int size)
    {
        var buffer =
            new List<T>(size);


        await foreach (
            var item in source)
        {
            buffer.Add(item);


            if (buffer.Count >= size)
            {
                yield return
                    buffer.ToArray();

                buffer.Clear();
            }
        }


        if (buffer.Count > 0)
        {
            yield return
                buffer.ToArray();
        }
    }
}
9. How the modules communicate

This is important.

Modules should not directly depend on each other's ViewModels.

Avoid:

ProductSearchViewModel
        ↓
new DataManagementViewModel()

or:

_dataManagementViewModel
    .LoadSomething();

Instead:

Module A
   │
   ▼
Shared Contract / Service
   │
   ▼
Module B

For navigation:

Module
   │
   ▼
INavigationService
   │
   ▼
Target Module

For shared data:

Module A
   │
   ▼
Shared Service / Repository / Cache
   │
   ▼
Module B

For loose communication, later we can introduce:

Module A
   │
   ▼
Event Aggregator / Message Bus
   │
   ▼
Module B
Final architecture
                         APP
                          │
                          ▼
                 Composition Root
                          │
                          ▼
                    DI Container
                          │
                          ▼
                      Shell
                          │
                 ┌────────┴────────┐
                 │                 │
                 ▼                 ▼
            Navigation         Content Region
                 │                 │
                 │                 ▼
                 │           Current Module
                 │                 │
                 ├─────────────────┤
                 │                 │
                 ▼                 ▼
          Product Search     Live Monitoring
                 │                 │
                 ▼                 ▼
           Debouncing        Throttling
                 │
                 ▼
          Data Management
                 │
          ┌──────┴──────┐
          ▼             ▼
        Paging        Caching
                 │
                 ▼
            Processing
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
     Batching  Priority  Timeout
                 │
                 ▼
             Reporting
                 │
        ┌────────┴────────┐
        ▼                 ▼
    Streaming          Chunking

This is the current recommended architecture direction: the application is segregated by feature/module, the Shell handles navigation, each module owns its View + ViewModel + module-specific logic, and shared infrastructure stays outside the modules.

is it layer based?

Not exactly. The current architecture is primarily feature/module-based, but inside each module there is some layering.

What we currently have
Modules
│
├── ProductSearch
│   ├── View
│   ├── ViewModel
│   └── Service
│
├── LiveMonitoring
│   ├── View
│   ├── ViewModel
│   └── Service
│
├── DataManagement
│   ├── View
│   ├── ViewModel
│   ├── Repository
│   └── Cache
│
└── Reporting
    ├── View
    ├── ViewModel
    └── Service

So this is better described as:

Feature-based / module-based architecture with layers inside each feature.

Pure layer-based architecture would look like this
Presentation
│
├── Views
│   ├── ProductSearchView
│   ├── LiveMonitoringView
│   └── ReportingView
│
└── ViewModels
    ├── ProductSearchViewModel
    ├── LiveMonitoringViewModel
    └── ReportingViewModel


Application / Business Layer
│
├── ProductSearchService
├── LiveMonitoringService
└── ReportService


Infrastructure / Data Layer
│
├── ProductRepository
├── Cache
└── External APIs

Here, everything is organized by technical responsibility.

All Views → one place
All ViewModels → one place
All Services → one place
All Repositories → one place

That is a classic layer-based architecture.

Our current architecture

Our architecture says:

ProductSearch
│
├── View
├── ViewModel
└── Service

Then:

DataManagement
│
├── View
├── ViewModel
├── Repository
└── Cache

So instead of grouping by technical type, we group by feature.

Layer-based
Views/
    AView
    BView
    CView

ViewModels/
    AViewModel
    BViewModel
    CViewModel

Services/
    AService
    BService
    CService
Feature-based
FeatureA/
    AView
    AViewModel
    AService

FeatureB/
    BView
    BViewModel
    BService

FeatureC/
    CView
    CViewModel
    CService
The important point

Our application is actually a hybrid:

┌─────────────────────────────────────┐
│ Application                         │
│                                     │
│  Feature / Module Organization      │
│                                     │
│  ├── Product Search                 │
│  │     ├── Presentation             │
│  │     └── Application Logic        │
│  │                                  │
│  ├── Data Management                │
│  │     ├── Presentation             │
│  │     └── Data Logic               │
│  │                                  │
│  └── Reporting                      │
│        ├── Presentation             │
│        └── Processing Logic         │
└─────────────────────────────────────┘

Within a feature, there is still a logical flow:

View
 ↓
ViewModel
 ↓
Service
 ↓
Repository / Infrastructure

So layering still exists, but the top-level project organization is feature/module-based.

Best way to describe the current architecture

The application uses a modular, feature-based architecture. Each module is internally organized into layers such as Presentation (View/ViewModel), Application/Service logic, and Infrastructure/Data access.

Or more simply:

Modules horizontally, layers vertically.

                    Product Search       Data Management
Presentation       View / ViewModel      View / ViewModel
Application        Search Service        Paging Service
Infrastructure     API/Data Source       Repository/Cache

That is probably the clearest model for what we are building.
