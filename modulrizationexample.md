# Modular WPF Monitoring Application
## Staff Engineer Interview Reference Implementation

> **Purpose:** A runnable/reference architecture for discussing WPF system design, modularization, MVVM, navigation, lifecycle management, streaming, buffering, backpressure, throttling, paging, caching, UI virtualization, cancellation, memory management, and testing.

---

# Table of Contents

1. Problem Scenario
2. Requirements and Assumptions
3. High-Level Architecture
4. Solution Structure
5. Scenario 1 — Modular Shell and Navigation
6. Scenario 2 — Navigation Lifecycle
7. Scenario 3 — Streaming and Producer/Consumer
8. Scenario 4 — Backpressure and Bounded Buffering
9. Scenario 5 — Throttled and Batched UI Updates
10. Scenario 6 — Module Communication and Event Bus
11. Scenario 7 — History Paging and Cancellation
12. Scenario 8 — Configuration Caching
13. Scenario 9 — UI Virtualization
14. Dependency Injection and Startup
15. Complete Code
16. Memory Leak and Lifecycle Considerations
17. Testing Strategy
18. Interview Walkthrough
19. Trade-offs and Alternatives

---

# 1. Problem Scenario

Assume the interviewer says:

> Design a WPF desktop application for monitoring devices. The application has multiple modules and receives a large amount of data from a backend or hardware system.

The application has five modules:

```text
Devices
Monitoring
Alarms
History
Configuration
```

The backend may:

- Send telemetry continuously.
- Produce data faster than the UI can render it.
- Contain millions of historical records.
- Return configuration data that is frequently read but rarely changed.

The user may also:

- Navigate quickly between modules.
- Change pages before the previous request completes.
- Leave the Monitoring screen while telemetry is still arriving.

The architecture must therefore handle:

```text
Modularity
Navigation
Lifecycle
Cancellation
Streaming
Buffering
Backpressure
Throttling
Batching
Paging
Caching
Virtualization
Memory management
Testing
```

---

# 2. Requirements and Assumptions

We use:

- .NET 8
- WPF
- MVVM
- Microsoft dependency injection / Generic Host
- `System.Threading.Channels`
- `IAsyncEnumerable<T>`
- `CancellationToken`

The architecture intentionally does **not** depend on Prism.

The reason is educational: we want to understand the concepts that a framework can later provide.

---

# 3. High-Level Architecture

```text
                           ┌────────────────────┐
                           │       Shell        │
                           │                    │
                           │ Navigation + Region│
                           └─────────┬──────────┘
                                     │
                             Navigation Service
                                     │
                     ┌───────────────┼───────────────┐
                     │               │               │
                     ▼               ▼               ▼
                  Routes          Lifecycle          DI
                     │               │
     ┌───────────────┼───────────────┼────────────────────┐
     ▼               ▼               ▼         ▼          ▼
  Device         Monitoring       Alarm      History   Configuration
     │               │               │         │          │
     │               ▼               │         ▼          ▼
     │         Telemetry Pipeline    │       Paging     Cache
     │               │               │
     │         Bounded Channel       │
     │               │               │
     │               ▼               ▼
     │          Processing ------> Event Bus
     │               │
     │               ▼
     │         UI Batch Buffer
     │               │
     └───────────────┼───────────────┘
                     ▼
              Backend / Hardware
```

A key design decision is that the UI is **not** the ingestion pipeline.

Bad:

```text
Backend
   ↓
ObservableCollection
   ↓
UI
```

Better:

```text
Backend
   ↓
Producer
   ↓
Bounded Channel
   ↓
Consumer / Processing
   ↓
UI Buffer
   ↓
Throttle / Batch
   ↓
UI
```

---

# 4. Solution Structure

For a real application, separate projects are useful:

```text
ModularMonitoringApp.sln
│
├── ModularMonitoringApp.Core
│   ├── Mvvm
│   ├── Navigation
│   ├── Lifecycle
│   ├── Messaging
│   ├── Models
│   └── Contracts
│
├── ModularMonitoringApp.Infrastructure
│   ├── Backend
│   ├── Streaming
│   ├── Caching
│   └── Messaging
│
└── ModularMonitoringApp.Wpf
    ├── Modules
    │   ├── Device
    │   ├── Monitoring
    │   ├── Alarm
    │   ├── History
    │   └── Configuration
    │
    ├── App.xaml
    ├── App.xaml.cs
    ├── MainWindow.xaml
    └── MainWindow.xaml.cs
```

For interview practice, this can initially be implemented in fewer projects and later split into separate assemblies.

---

# 5. Scenario 1 — Modular Shell and Navigation

## Interview Question

> We have five modules. How do you navigate between them without the Shell knowing the internal implementation of every module?

We introduce a route abstraction.

```text
User Click
    ↓
Command
    ↓
INavigationService
    ↓
Route Registry
    ↓
Resolve ViewModel
    ↓
Navigation Lifecycle
    ↓
Shell.CurrentViewModel
    ↓
DataTemplate
    ↓
View
```

---

## 5.1 ObservableObject

### `Core/Mvvm/ObservableObject.cs`

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace ModularMonitoringApp.Core.Mvvm;

public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected bool SetProperty<T>(
        ref T field,
        T value,
        [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;

        field = value;

        PropertyChanged?.Invoke(
            this,
            new PropertyChangedEventArgs(propertyName));

        return true;
    }

    protected void OnPropertyChanged(
        [CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(
            this,
            new PropertyChangedEventArgs(propertyName));
    }
}
```

### Why?

The ViewModel must notify WPF when a property changes.

```text
ViewModel Property
        ↓
PropertyChanged
        ↓
Binding Engine
        ↓
UI updates
```

---

## 5.2 RelayCommand

### `Core/Mvvm/RelayCommand.cs`

```csharp
using System.Windows.Input;

namespace ModularMonitoringApp.Core.Mvvm;

public sealed class RelayCommand : ICommand
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

    public bool CanExecute(object? parameter)
        => _canExecute?.Invoke() ?? true;

    public void Execute(object? parameter)
        => _execute();

    public void RaiseCanExecuteChanged()
        => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

---

## 5.3 AsyncRelayCommand

### `Core/Mvvm/AsyncRelayCommand.cs`

```csharp
using System.Windows.Input;

namespace ModularMonitoringApp.Core.Mvvm;

public sealed class AsyncRelayCommand : ICommand
{
    private readonly Func<CancellationToken, Task> _execute;
    private readonly Func<bool>? _canExecute;

    private bool _isExecuting;
    private CancellationTokenSource? _cts;

    public AsyncRelayCommand(
        Func<CancellationToken, Task> execute,
        Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool IsExecuting => _isExecuting;

    public event EventHandler? CanExecuteChanged;

    public bool CanExecute(object? parameter)
        => !_isExecuting &&
           (_canExecute?.Invoke() ?? true);

    public async void Execute(object? parameter)
    {
        if (!CanExecute(parameter))
            return;

        _cts?.Cancel();
        _cts?.Dispose();

        _cts = new CancellationTokenSource();

        try
        {
            _isExecuting = true;
            RaiseCanExecuteChanged();

            await _execute(_cts.Token);
        }
        catch (OperationCanceledException)
        {
            // Expected when the operation is cancelled.
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }

    public void Cancel()
        => _cts?.Cancel();

    public void RaiseCanExecuteChanged()
        => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

### Why cancellation?

A UI operation may become irrelevant.

Example:

```text
User requests Page 10
        ↓
Backend request starts
        ↓
User immediately requests Page 20
        ↓
Page 10 is now stale
        ↓
Cancel Page 10
```

---

## 5.4 Navigation Lifecycle Contract

### `Core/Lifecycle/INavigationAware.cs`

```csharp
namespace ModularMonitoringApp.Core.Lifecycle;

public interface INavigationAware
{
    Task OnNavigatedToAsync(
        CancellationToken cancellationToken);

    Task OnNavigatedFromAsync(
        CancellationToken cancellationToken);
}
```

This is the first important improvement over a simple navigation service.

A ViewModel can now say:

```text
When I become active:
    Start or resume required work.

When I become inactive:
    Stop or cancel unnecessary work.
```

---

## 5.5 Route Registry

### `Core/Navigation/IRouteRegistry.cs`

```csharp
namespace ModularMonitoringApp.Core.Navigation;

public interface IRouteRegistry
{
    void Register(string route, Type viewModelType);

    Type GetViewModelType(string route);
}
```

### `Core/Navigation/RouteRegistry.cs`

```csharp
namespace ModularMonitoringApp.Core.Navigation;

public sealed class RouteRegistry : IRouteRegistry
{
    private readonly Dictionary<string, Type> _routes =
        new(StringComparer.OrdinalIgnoreCase);

    public void Register(
        string route,
        Type viewModelType)
    {
        _routes[route] = viewModelType;
    }

    public Type GetViewModelType(string route)
    {
        if (!_routes.TryGetValue(route, out var type))
        {
            throw new InvalidOperationException(
                $"Route '{route}' is not registered.");
        }

        return type;
    }
}
```

The Shell does not need:

```csharp
if (route == "monitoring")
{
    CurrentViewModel = new MonitoringViewModel(...);
}
```

That would tightly couple the Shell to every module.

Instead:

```text
Shell
  ↓
"monitoring"
  ↓
Route Registry
  ↓
MonitoringViewModel
```

---

## 5.6 Navigation Host

### `Core/Navigation/IShellNavigationHost.cs`

```csharp
namespace ModularMonitoringApp.Core.Navigation;

public interface IShellNavigationHost
{
    object? CurrentViewModel { get; set; }
}
```

---

## 5.7 Navigation Service with Lifecycle

### `Core/Navigation/INavigationService.cs`

```csharp
namespace ModularMonitoringApp.Core.Navigation;

public interface INavigationService
{
    Task NavigateAsync(
        string route,
        CancellationToken cancellationToken = default);
}
```

### `Core/Navigation/NavigationService.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using ModularMonitoringApp.Core.Lifecycle;

namespace ModularMonitoringApp.Core.Navigation;

public sealed class NavigationService : INavigationService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly IRouteRegistry _routes;
    private readonly IShellNavigationHost _shell;

    private CancellationTokenSource? _navigationCts;

    public NavigationService(
        IServiceProvider serviceProvider,
        IRouteRegistry routes,
        IShellNavigationHost shell)
    {
        _serviceProvider = serviceProvider;
        _routes = routes;
        _shell = shell;
    }

    public async Task NavigateAsync(
        string route,
        CancellationToken cancellationToken = default)
    {
        _navigationCts?.Cancel();
        _navigationCts?.Dispose();

        _navigationCts =
            CancellationTokenSource.CreateLinkedTokenSource(
                cancellationToken);

        var token = _navigationCts.Token;

        var oldViewModel = _shell.CurrentViewModel;

        if (oldViewModel is INavigationAware oldAware)
        {
            await oldAware.OnNavigatedFromAsync(token);
        }

        token.ThrowIfCancellationRequested();

        var type = _routes.GetViewModelType(route);

        var newViewModel =
            _serviceProvider.GetRequiredService(type);

        _shell.CurrentViewModel = newViewModel;

        if (newViewModel is INavigationAware newAware)
        {
            await newAware.OnNavigatedToAsync(token);
        }
    }
}
```

## Important lifecycle discussion

The sequence is:

```text
Current ViewModel
        ↓
OnNavigatedFromAsync
        ↓
Cancel unnecessary work
        ↓
Resolve next ViewModel
        ↓
Set CurrentViewModel
        ↓
OnNavigatedToAsync
        ↓
Start necessary work
```

In a more advanced production router, you might add:

- navigation guards
- navigation parameters
- rollback when activation fails
- per-view cancellation tokens
- scoped lifetimes
- navigation journal/back stack

For interview practice, this version makes the lifecycle concept explicit.

---

# 6. Shell

## `ShellViewModel.cs`

```csharp
using System.Windows.Input;
using ModularMonitoringApp.Core.Mvvm;
using ModularMonitoringApp.Core.Navigation;

namespace ModularMonitoringApp.Wpf;

public sealed class ShellViewModel :
    ObservableObject,
    IShellNavigationHost
{
    private object? _currentViewModel;

    public object? CurrentViewModel
    {
        get => _currentViewModel;
        set => SetProperty(ref _currentViewModel, value);
    }

    public ICommand NavigateDevicesCommand { get; }
    public ICommand NavigateMonitoringCommand { get; }
    public ICommand NavigateAlarmsCommand { get; }
    public ICommand NavigateHistoryCommand { get; }
    public ICommand NavigateConfigurationCommand { get; }

    public ShellViewModel(
        INavigationService navigation)
    {
        NavigateDevicesCommand =
            new AsyncRelayCommand(
                token => navigation.NavigateAsync(
                    "devices", token));

        NavigateMonitoringCommand =
            new AsyncRelayCommand(
                token => navigation.NavigateAsync(
                    "monitoring", token));

        NavigateAlarmsCommand =
            new AsyncRelayCommand(
                token => navigation.NavigateAsync(
                    "alarms", token));

        NavigateHistoryCommand =
            new AsyncRelayCommand(
                token => navigation.NavigateAsync(
                    "history", token));

        NavigateConfigurationCommand =
            new AsyncRelayCommand(
                token => navigation.NavigateAsync(
                    "configuration", token));
    }
}
```

---

# 7. Main Window

## `MainWindow.xaml`

```xml
<Window
    x:Class="ModularMonitoringApp.Wpf.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Title="Modular Monitoring Application"
    Height="700"
    Width="1200">

    <Grid>

        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="180"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>

        <StackPanel Grid.Column="0">

            <Button
                Margin="8"
                Content="Devices"
                Command="{Binding NavigateDevicesCommand}" />

            <Button
                Margin="8"
                Content="Monitoring"
                Command="{Binding NavigateMonitoringCommand}" />

            <Button
                Margin="8"
                Content="Alarms"
                Command="{Binding NavigateAlarmsCommand}" />

            <Button
                Margin="8"
                Content="History"
                Command="{Binding NavigateHistoryCommand}" />

            <Button
                Margin="8"
                Content="Configuration"
                Command="{Binding NavigateConfigurationCommand}" />

        </StackPanel>

        <ContentControl
            Grid.Column="1"
            Margin="10"
            Content="{Binding CurrentViewModel}" />

    </Grid>
</Window>
```

---

## `MainWindow.xaml.cs`

```csharp
using System.Windows;

namespace ModularMonitoringApp.Wpf;

public partial class MainWindow : Window
{
    public MainWindow(ShellViewModel viewModel)
    {
        InitializeComponent();

        DataContext = viewModel;
    }
}
```

---

# 8. Scenario 2 — Continuous Telemetry

## Interview Question

> The backend continuously sends telemetry. How would you prevent the UI from becoming the bottleneck?

The first rule:

```text
Do not do this:

Backend
    ↓
ObservableCollection.Add()
    ↓
UI
```

Instead:

```text
Backend
   ↓
Producer
   ↓
Channel<T>
   ↓
Consumer
   ↓
Business Processing
   ↓
UI Buffer
   ↓
Periodic Batch
   ↓
Dispatcher
   ↓
ObservableCollection
```

---

# 9. Telemetry Model

### `Core/Models/TelemetryData.cs`

```csharp
namespace ModularMonitoringApp.Core.Models;

public sealed record TelemetryData(
    string DeviceId,
    DateTime Timestamp,
    double Temperature,
    double Pressure);
```

---

# 10. Backend Contract

### `Core/Contracts/ITelemetryBackend.cs`

```csharp
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Core.Contracts;

public interface ITelemetryBackend
{
    IAsyncEnumerable<TelemetryData> StreamAsync(
        CancellationToken cancellationToken);
}
```

---

# 11. Fake Backend

### `Infrastructure/Backend/FakeTelemetryBackend.cs`

```csharp
using System.Runtime.CompilerServices;
using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Infrastructure.Backend;

public sealed class FakeTelemetryBackend :
    ITelemetryBackend
{
    public async IAsyncEnumerable<TelemetryData> StreamAsync(
        [EnumeratorCancellation]
        CancellationToken cancellationToken)
    {
        var random = new Random();

        while (!cancellationToken.IsCancellationRequested)
        {
            yield return new TelemetryData(
                DeviceId: $"Device-{random.Next(1, 6)}",
                Timestamp: DateTime.UtcNow,
                Temperature: random.NextDouble() * 120,
                Pressure: random.NextDouble() * 50);

            await Task.Delay(
                TimeSpan.FromMilliseconds(1),
                cancellationToken);
        }
    }
}
```

This simulates approximately 1000 messages per second.

In a real application the source might instead be:

- WebSocket
- gRPC stream
- TCP socket
- serial communication
- hardware SDK
- message broker

The pipeline should not need to care where the data originated.

---

# 12. Scenario 3 — Producer Faster Than Consumer

## The problem

Suppose:

```text
Producer = 10,000 messages/sec
Consumer = 2,000 messages/sec
```

Without limits:

```text
Producer
   ↓
Queue grows forever
   ↓
Memory grows
   ↓
OutOfMemoryException
```

Use a bounded buffer.

---

# 13. TelemetryChannel

### `Infrastructure/Streaming/TelemetryChannel.cs`

```csharp
using System.Threading.Channels;
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Infrastructure.Streaming;

public sealed class TelemetryChannel
{
    private readonly Channel<TelemetryData> _channel;

    public TelemetryChannel()
    {
        _channel = Channel.CreateBounded<TelemetryData>(
            new BoundedChannelOptions(10_000)
            {
                SingleWriter = true,
                SingleReader = true,
                FullMode =
                    BoundedChannelFullMode.DropOldest
            });
    }

    public ChannelWriter<TelemetryData> Writer
        => _channel.Writer;

    public ChannelReader<TelemetryData> Reader
        => _channel.Reader;
}
```

## Why bounded?

Because memory is a resource that must have a limit.

The system is now saying:

```text
Maximum buffered telemetry = 10,000
```

That creates a known backpressure policy.

---

## Choosing a full mode

### `Wait`

```text
Producer waits
```

Use when every message matters and slowing the producer is acceptable.

### `DropOldest`

```text
Discard old data
Keep recent data
```

Often useful for a live dashboard.

### `DropNewest`

Discard the most recent buffered item.

### `DropWrite`

Discard the incoming item.

The correct choice depends on business requirements.

A Staff-level answer should explicitly say:

> I cannot choose the dropping policy without understanding whether telemetry is only for visualization or is business-critical data that must never be lost.

---

# 14. Telemetry Producer

### `Infrastructure/Streaming/TelemetryProducer.cs`

```csharp
using ModularMonitoringApp.Core.Contracts;

namespace ModularMonitoringApp.Infrastructure.Streaming;

public sealed class TelemetryProducer
{
    private readonly ITelemetryBackend _backend;
    private readonly TelemetryChannel _channel;

    public TelemetryProducer(
        ITelemetryBackend backend,
        TelemetryChannel channel)
    {
        _backend = backend;
        _channel = channel;
    }

    public async Task RunAsync(
        CancellationToken cancellationToken)
    {
        try
        {
            await foreach (
                var telemetry in _backend.StreamAsync(
                    cancellationToken))
            {
                await _channel.Writer.WriteAsync(
                    telemetry,
                    cancellationToken);
            }
        }
        catch (OperationCanceledException)
        {
            // Normal shutdown.
        }
        finally
        {
            _channel.Writer.TryComplete();
        }
    }
}
```

---

# 15. Important Design Note: Who Owns the Stream?

This is an important interview question.

A naive design makes `MonitoringViewModel` own the producer.

```text
Monitoring ViewModel
        ↓
Connect to Backend
        ↓
Read Stream
```

That works for a small application but becomes questionable when:

- other modules need the same stream
- the stream should continue while the user navigates away
- reconnection should be centralized
- multiple subscribers need processed state

A better design can be:

```text
Application-level stream service
        ↓
TelemetryChannel
        ↓
Consumers
```

Then the Monitoring module controls only whether it is currently displaying data.

For this reference implementation, the producer is treated as infrastructure and can be started once at application startup.

---

# 16. Scenario 4 — Throttled UI Updates

Suppose the application receives:

```text
10,000 telemetry events/sec
```

The UI does not need 10,000 redraws per second.

We separate:

```text
Ingestion frequency
        ↓
Processing frequency
        ↓
UI refresh frequency
```

Example:

```text
10,000 events/sec processed
        ↓
UI updated 5 times/sec
```

---

# 17. MonitoringViewModel

### `Wpf/Modules/Monitoring/MonitoringViewModel.cs`

```csharp
using System.Collections.Concurrent;
using System.Collections.ObjectModel;
using System.Windows;
using ModularMonitoringApp.Core.Lifecycle;
using ModularMonitoringApp.Core.Mvvm;
using ModularMonitoringApp.Core.Models;
using ModularMonitoringApp.Infrastructure.Streaming;

namespace ModularMonitoringApp.Wpf.Modules.Monitoring;

public sealed class MonitoringViewModel :
    ObservableObject,
    INavigationAware,
    IDisposable
{
    private readonly TelemetryChannel _channel;

    private readonly ConcurrentQueue<TelemetryData>
        _pendingUiItems = new();

    private CancellationTokenSource? _lifecycleCts;

    private Task? _consumerTask;
    private Task? _uiUpdateTask;

    private bool _isActive;

    public bool IsActive
    {
        get => _isActive;
        private set => SetProperty(ref _isActive, value);
    }

    public ObservableCollection<TelemetryData> Items { get; }
        = new();

    public MonitoringViewModel(
        TelemetryChannel channel)
    {
        _channel = channel;
    }

    public Task OnNavigatedToAsync(
        CancellationToken cancellationToken)
    {
        if (IsActive)
            return Task.CompletedTask;

        IsActive = true;

        _lifecycleCts =
            CancellationTokenSource.CreateLinkedTokenSource(
                cancellationToken);

        var token = _lifecycleCts.Token;

        _consumerTask =
            ConsumeAsync(token);

        _uiUpdateTask =
            UpdateUiAsync(token);

        return Task.CompletedTask;
    }

    public async Task OnNavigatedFromAsync(
        CancellationToken cancellationToken)
    {
        if (!IsActive)
            return;

        IsActive = false;

        _lifecycleCts?.Cancel();

        try
        {
            var tasks =
                new[] { _consumerTask, _uiUpdateTask }
                .Where(x => x is not null)
                .Cast<Task>();

            await Task.WhenAll(tasks);
        }
        catch (OperationCanceledException)
        {
            // Expected.
        }
        finally
        {
            _consumerTask = null;
            _uiUpdateTask = null;

            _lifecycleCts?.Dispose();
            _lifecycleCts = null;
        }
    }

    private async Task ConsumeAsync(
        CancellationToken cancellationToken)
    {
        try
        {
            await foreach (
                var telemetry in
                _channel.Reader.ReadAllAsync(
                    cancellationToken))
            {
                ProcessTelemetry(telemetry);

                _pendingUiItems.Enqueue(telemetry);
            }
        }
        catch (OperationCanceledException)
        {
            // Expected.
        }
    }

    private void ProcessTelemetry(
        TelemetryData telemetry)
    {
        // Business processing belongs here.
        //
        // Examples:
        // - Update latest device state
        // - Calculate statistics
        // - Detect abnormal values
        // - Publish an event
    }

    private async Task UpdateUiAsync(
        CancellationToken cancellationToken)
    {
        using var timer =
            new PeriodicTimer(
                TimeSpan.FromMilliseconds(200));

        try
        {
            while (await timer.WaitForNextTickAsync(
                cancellationToken))
            {
                var batch = new List<TelemetryData>();

                while (_pendingUiItems.TryDequeue(
                    out var item))
                {
                    batch.Add(item);
                }

                if (batch.Count == 0)
                    continue;

                await Application.Current.Dispatcher
                    .InvokeAsync(
                        () => AddBatchToUi(batch),
                        System.Windows.Threading.DispatcherPriority.Background,
                        cancellationToken);
            }
        }
        catch (OperationCanceledException)
        {
            // Expected.
        }
    }

    private void AddBatchToUi(
        IReadOnlyList<TelemetryData> batch)
    {
        foreach (var item in batch)
        {
            Items.Add(item);
        }

        const int MaximumVisibleItems = 2_000;

        while (Items.Count > MaximumVisibleItems)
        {
            Items.RemoveAt(0);
        }
    }

    public void Dispose()
    {
        _lifecycleCts?.Cancel();
        _lifecycleCts?.Dispose();
    }
}
```

---

# 18. What Happens When the User Navigates Away?

This is the important scenario.

```text
Monitoring screen active
        ↓
Consumer is reading
UI timer is updating
        ↓
User clicks History
        ↓
Navigation Service
        ↓
OnNavigatedFromAsync()
        ↓
CancellationTokenSource.Cancel()
        ↓
Consumer exits
UI timer exits
        ↓
Tasks complete
        ↓
History becomes active
```

This avoids background work continuing unnecessarily.

## Important distinction

There are two possible requirements.

### Requirement A — Stop the stream when the user leaves

Then the Monitoring module can own the stream lifecycle.

### Requirement B — The application should continue receiving data

Then the stream should be owned by an application-level service, and the Monitoring ViewModel should only subscribe/display while active.

This reference uses the second conceptual model: the infrastructure pipeline can remain active while the view lifecycle controls consumption/display.

---

# 19. Scenario 5 — Module Communication

## Interview Question

> Monitoring detects a high temperature. How does the Alarm module know?

Avoid:

```text
MonitoringViewModel
    ↓
AlarmViewModel
```

Instead:

```text
Monitoring
    ↓
Publish AlarmRaised
    ↓
Event Bus
    ↓
Alarm Module
```

---

# 20. Event Bus Contract

### `Core/Messaging/IEventBus.cs`

```csharp
namespace ModularMonitoringApp.Core.Messaging;

public interface IEventBus
{
    IDisposable Subscribe<TEvent>(
        Action<TEvent> handler);

    void Publish<TEvent>(
        TEvent @event);
}
```

---

# 21. Event Bus Implementation

### `Infrastructure/Messaging/EventBus.cs`

```csharp
using System.Collections.Concurrent;
using ModularMonitoringApp.Core.Messaging;

namespace ModularMonitoringApp.Infrastructure.Messaging;

public sealed class EventBus : IEventBus
{
    private readonly ConcurrentDictionary<
        Type,
        List<Delegate>> _handlers = new();

    public IDisposable Subscribe<TEvent>(
        Action<TEvent> handler)
    {
        var handlers =
            _handlers.GetOrAdd(
                typeof(TEvent),
                _ => new List<Delegate>());

        lock (handlers)
        {
            handlers.Add(handler);
        }

        return new Subscription(
            () =>
            {
                lock (handlers)
                {
                    handlers.Remove(handler);
                }
            });
    }

    public void Publish<TEvent>(
        TEvent @event)
    {
        if (!_handlers.TryGetValue(
                typeof(TEvent),
                out var handlers))
            return;

        Delegate[] snapshot;

        lock (handlers)
        {
            snapshot = handlers.ToArray();
        }

        foreach (var handler in snapshot)
        {
            ((Action<TEvent>)handler).Invoke(@event);
        }
    }

    private sealed class Subscription : IDisposable
    {
        private Action? _unsubscribe;

        public Subscription(Action unsubscribe)
        {
            _unsubscribe = unsubscribe;
        }

        public void Dispose()
        {
            Interlocked.Exchange(
                ref _unsubscribe,
                null)?.Invoke();
        }
    }
}
```

---

# 22. Alarm Model

### `Core/Models/AlarmRaised.cs`

```csharp
namespace ModularMonitoringApp.Core.Models;

public sealed record AlarmRaised(
    string DeviceId,
    string Message,
    DateTime Timestamp);
```

---

# 23. Publish Alarm

Add an `IEventBus` dependency to `MonitoringViewModel`.

```csharp
private readonly IEventBus _eventBus;
```

Constructor:

```csharp
public MonitoringViewModel(
    TelemetryChannel channel,
    IEventBus eventBus)
{
    _channel = channel;
    _eventBus = eventBus;
}
```

Processing:

```csharp
private void ProcessTelemetry(
    TelemetryData telemetry)
{
    if (telemetry.Temperature > 90)
    {
        _eventBus.Publish(
            new AlarmRaised(
                telemetry.DeviceId,
                $"High temperature: {telemetry.Temperature:F1}",
                DateTime.UtcNow));
    }
}
```

---

# 24. AlarmViewModel with Lifecycle Subscription

A common memory leak occurs when an event bus retains a ViewModel through a delegate.

Therefore, subscribe while active and unsubscribe when inactive.

### `Wpf/Modules/Alarm/AlarmViewModel.cs`

```csharp
using System.Collections.ObjectModel;
using System.Windows;
using ModularMonitoringApp.Core.Lifecycle;
using ModularMonitoringApp.Core.Messaging;
using ModularMonitoringApp.Core.Models;
using ModularMonitoringApp.Core.Mvvm;

namespace ModularMonitoringApp.Wpf.Modules.Alarm;

public sealed class AlarmViewModel :
    ObservableObject,
    INavigationAware,
    IDisposable
{
    private readonly IEventBus _eventBus;

    private IDisposable? _subscription;

    public ObservableCollection<AlarmRaised> Alarms { get; }
        = new();

    public AlarmViewModel(
        IEventBus eventBus)
    {
        _eventBus = eventBus;
    }

    public Task OnNavigatedToAsync(
        CancellationToken cancellationToken)
    {
        _subscription ??=
            _eventBus.Subscribe<AlarmRaised>(
                OnAlarmRaised);

        return Task.CompletedTask;
    }

    public Task OnNavigatedFromAsync(
        CancellationToken cancellationToken)
    {
        _subscription?.Dispose();
        _subscription = null;

        return Task.CompletedTask;
    }

    private void OnAlarmRaised(
        AlarmRaised alarm)
    {
        Application.Current.Dispatcher.InvokeAsync(
            () => Alarms.Insert(0, alarm));
    }

    public void Dispose()
    {
        _subscription?.Dispose();
    }
}
```

---

# 25. Memory Leak Explanation

A singleton event bus may hold:

```text
Event Bus
    │
    ▼
Delegate
    │
    ▼
AlarmViewModel
```

If the ViewModel is no longer needed but remains subscribed:

```text
Garbage Collector
    ↓
Cannot collect ViewModel
    ↓
Memory leak
```

The lifecycle subscription fixes this:

```text
OnNavigatedTo
    ↓
Subscribe

OnNavigatedFrom
    ↓
Dispose subscription
```

Alternative approaches:

- weak events
- weak references
- scoped message bus
- explicit disposal

The best choice depends on ownership and lifetime.

---

# 26. Scenario 6 — History with Millions of Records

## Interview Question

> There are 10 million historical records. How do you display them?

Do not:

```csharp
var records = await backend.GetAllAsync();
Items = new ObservableCollection<TelemetryData>(records);
```

Instead:

```text
User requests page
        ↓
Cancel previous request
        ↓
Check page cache
        │
    ┌───┴────┐
   Hit      Miss
    │         │
    │      Backend
    │         │
    └────┬────┘
         ▼
       Display
```

---

# 27. Paging Models

### `Core/Models/PageRequest.cs`

```csharp
namespace ModularMonitoringApp.Core.Models;

public sealed record PageRequest(
    int PageNumber,
    int PageSize);
```

### `Core/Models/PageResult.cs`

```csharp
namespace ModularMonitoringApp.Core.Models;

public sealed record PageResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount);
```

---

# 28. History Backend

### `Core/Contracts/IHistoryBackend.cs`

```csharp
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Core.Contracts;

public interface IHistoryBackend
{
    Task<PageResult<TelemetryData>> GetPageAsync(
        PageRequest request,
        CancellationToken cancellationToken);
}
```

### `Infrastructure/Backend/FakeHistoryBackend.cs`

```csharp
using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Infrastructure.Backend;

public sealed class FakeHistoryBackend :
    IHistoryBackend
{
    private const int TotalRecords = 10_000_000;

    public async Task<PageResult<TelemetryData>>
        GetPageAsync(
            PageRequest request,
            CancellationToken cancellationToken)
    {
        await Task.Delay(
            300,
            cancellationToken);

        var random = new Random();

        var items =
            Enumerable.Range(
                0,
                request.PageSize)
            .Select(index =>
            {
                var absoluteIndex =
                    request.PageNumber *
                    request.PageSize +
                    index;

                return new TelemetryData(
                    $"Device-{absoluteIndex % 5}",
                    DateTime.UtcNow.AddMinutes(
                        -absoluteIndex),
                    random.NextDouble() * 100,
                    random.NextDouble() * 50);
            })
            .ToList();

        return new PageResult<TelemetryData>(
            items,
            TotalRecords);
    }
}
```

---

# 29. Page Cache

### `Core/Contracts/IPageCache.cs`

```csharp
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Core.Contracts;

public interface IPageCache<T>
{
    bool TryGet(
        int pageNumber,
        out PageResult<T>? result);

    void Set(
        int pageNumber,
        PageResult<T> result);
}
```

### `Infrastructure/Caching/PageCache.cs`

```csharp
using System.Collections.Concurrent;
using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Infrastructure.Caching;

public sealed class PageCache<T> :
    IPageCache<T>
{
    private readonly ConcurrentDictionary<
        int,
        PageResult<T>> _cache = new();

    public bool TryGet(
        int pageNumber,
        out PageResult<T>? result)
    {
        return _cache.TryGetValue(
            pageNumber,
            out result);
    }

    public void Set(
        int pageNumber,
        PageResult<T> result)
    {
        _cache[pageNumber] = result;
    }
}
```

## Production concern

This simple cache grows forever.

For production, consider:

- maximum page count
- LRU eviction
- TTL
- memory budget
- prefetch policy

The example is intentionally simple so the interview discussion can evolve.

---

# 30. HistoryViewModel with Cancellation

### `Wpf/Modules/History/HistoryViewModel.cs`

```csharp
using System.Collections.ObjectModel;
using System.Windows.Input;
using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Lifecycle;
using ModularMonitoringApp.Core.Models;
using ModularMonitoringApp.Core.Mvvm;

namespace ModularMonitoringApp.Wpf.Modules.History;

public sealed class HistoryViewModel :
    ObservableObject,
    INavigationAware,
    IDisposable
{
    private readonly IHistoryBackend _backend;
    private readonly IPageCache<TelemetryData> _cache;

    private CancellationTokenSource? _requestCts;

    private int _currentPage;

    public int CurrentPage
    {
        get => _currentPage;
        private set => SetProperty(
            ref _currentPage,
            value);
    }

    public ObservableCollection<TelemetryData> Items { get; }
        = new();

    public ICommand NextPageCommand { get; }
    public ICommand PreviousPageCommand { get; }

    public HistoryViewModel(
        IHistoryBackend backend,
        IPageCache<TelemetryData> cache)
    {
        _backend = backend;
        _cache = cache;

        NextPageCommand =
            new AsyncRelayCommand(
                token => LoadPageAsync(
                    CurrentPage + 1,
                    token));

        PreviousPageCommand =
            new AsyncRelayCommand(
                token => LoadPageAsync(
                    Math.Max(CurrentPage - 1, 0),
                    token));
    }

    public async Task OnNavigatedToAsync(
        CancellationToken cancellationToken)
    {
        if (Items.Count == 0)
        {
            await LoadPageAsync(
                0,
                cancellationToken);
        }
    }

    public Task OnNavigatedFromAsync(
        CancellationToken cancellationToken)
    {
        _requestCts?.Cancel();

        return Task.CompletedTask;
    }

    public async Task LoadPageAsync(
        int pageNumber,
        CancellationToken outerToken)
    {
        _requestCts?.Cancel();
        _requestCts?.Dispose();

        _requestCts =
            CancellationTokenSource.CreateLinkedTokenSource(
                outerToken);

        var token = _requestCts.Token;

        try
        {
            PageResult<TelemetryData>? page;

            if (!_cache.TryGet(
                    pageNumber,
                    out page))
            {
                page = await _backend.GetPageAsync(
                    new PageRequest(
                        pageNumber,
                        100),
                    token);

                _cache.Set(
                    pageNumber,
                    page);
            }

            token.ThrowIfCancellationRequested();

            Items.Clear();

            foreach (var item in page.Items)
            {
                Items.Add(item);
            }

            CurrentPage = pageNumber;
        }
        catch (OperationCanceledException)
        {
            // Stale request.
        }
    }

    public void Dispose()
    {
        _requestCts?.Cancel();
        _requestCts?.Dispose();
    }
}
```

---

# 31. Why Cancellation Matters

Without cancellation:

```text
Request Page 10 ───────────────┐
                                │
Request Page 20 ────────┐       │
                         │       │
Page 20 returns          ▼       │
UI shows Page 20                 │
                                 ▼
Page 10 returns
UI accidentally shows Page 10
```

With cancellation:

```text
Request Page 10
        ↓
User requests Page 20
        ↓
Cancel Page 10
        ↓
Load Page 20
        ↓
Only current request can update UI
```

---

# 32. Scenario 7 — Configuration Caching

## Problem

Configuration is:

- expensive to retrieve
- requested frequently
- changed infrequently

Use:

```text
Request
   ↓
Cache
   │
Hit ───► Return

Miss
   ↓
Backend
   ↓
Cache
   ↓
Return
```

---

# 33. Configuration Models

### `Core/Models/DeviceConfiguration.cs`

```csharp
namespace ModularMonitoringApp.Core.Models;

public sealed record DeviceConfiguration(
    double AlarmThreshold,
    int SamplingRate,
    string DisplayMode);
```

---

# 34. Configuration Contracts

```csharp
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Core.Contracts;

public interface IConfigurationBackend
{
    Task<DeviceConfiguration> GetAsync(
        CancellationToken cancellationToken);

    Task UpdateAsync(
        DeviceConfiguration configuration,
        CancellationToken cancellationToken);
}

public interface IConfigurationService
{
    Task<DeviceConfiguration> GetAsync(
        CancellationToken cancellationToken);

    Task UpdateAsync(
        DeviceConfiguration configuration,
        CancellationToken cancellationToken);
}
```

---

# 35. CachedConfigurationService

### `Infrastructure/Caching/CachedConfigurationService.cs`

```csharp
using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Models;

namespace ModularMonitoringApp.Infrastructure.Caching;

public sealed class CachedConfigurationService :
    IConfigurationService
{
    private readonly IConfigurationBackend _backend;

    private readonly SemaphoreSlim _lock =
        new(1, 1);

    private DeviceConfiguration? _cached;

    public CachedConfigurationService(
        IConfigurationBackend backend)
    {
        _backend = backend;
    }

    public async Task<DeviceConfiguration> GetAsync(
        CancellationToken cancellationToken)
    {
        if (_cached is not null)
            return _cached;

        await _lock.WaitAsync(cancellationToken);

        try
        {
            if (_cached is not null)
                return _cached;

            _cached =
                await _backend.GetAsync(
                    cancellationToken);

            return _cached;
        }
        finally
        {
            _lock.Release();
        }
    }

    public async Task UpdateAsync(
        DeviceConfiguration configuration,
        CancellationToken cancellationToken)
    {
        await _backend.UpdateAsync(
            configuration,
            cancellationToken);

        // Update cache only after backend success.
        _cached = configuration;
    }
}
```

---

# 36. Why the SemaphoreSlim?

Suppose five ViewModels request configuration simultaneously.

Without synchronization:

```text
Request 1 → Backend
Request 2 → Backend
Request 3 → Backend
Request 4 → Backend
Request 5 → Backend
```

With synchronization:

```text
Request 1
    ↓
Backend
    ↓
Cache filled

Requests 2–5
    ↓
Read cache
```

This is a simple example of avoiding a cache stampede.

---

# 37. Scenario 8 — UI Virtualization

## `HistoryView.xaml`

```xml
<UserControl
    x:Class="ModularMonitoringApp.Wpf.Modules.History.HistoryView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <DockPanel>

        <StackPanel
            DockPanel.Dock="Top"
            Orientation="Horizontal">

            <Button
                Content="Previous"
                Command="{Binding PreviousPageCommand}" />

            <TextBlock
                Margin="10"
                VerticalAlignment="Center"
                Text="{Binding CurrentPage}" />

            <Button
                Content="Next"
                Command="{Binding NextPageCommand}" />

        </StackPanel>

        <ListView
            ItemsSource="{Binding Items}"
            ScrollViewer.CanContentScroll="True"
            VirtualizingPanel.IsVirtualizing="True"
            VirtualizingPanel.VirtualizationMode="Recycling">

            <ListView.View>

                <GridView>

                    <GridViewColumn
                        Header="Device"
                        DisplayMemberBinding="{Binding DeviceId}" />

                    <GridViewColumn
                        Header="Timestamp"
                        DisplayMemberBinding="{Binding Timestamp}" />

                    <GridViewColumn
                        Header="Temperature"
                        DisplayMemberBinding="{Binding Temperature}" />

                    <GridViewColumn
                        Header="Pressure"
                        DisplayMemberBinding="{Binding Pressure}" />

                </GridView>

            </ListView.View>

        </ListView>

    </DockPanel>

</UserControl>
```

## Important distinction

```text
Paging
    ↓
Controls how much DATA is loaded.

UI Virtualization
    ↓
Controls how many UI containers are created.
```

They solve different problems.

---

# 38. DataTemplates

The Shell stores a ViewModel.

WPF chooses the View.

## `App.xaml`

```xml
<Application
    x:Class="ModularMonitoringApp.Wpf.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:deviceVm="clr-namespace:ModularMonitoringApp.Wpf.Modules.Device"
    xmlns:deviceView="clr-namespace:ModularMonitoringApp.Wpf.Modules.Device"
    xmlns:monitorVm="clr-namespace:ModularMonitoringApp.Wpf.Modules.Monitoring"
    xmlns:monitorView="clr-namespace:ModularMonitoringApp.Wpf.Modules.Monitoring"
    xmlns:alarmVm="clr-namespace:ModularMonitoringApp.Wpf.Modules.Alarm"
    xmlns:alarmView="clr-namespace:ModularMonitoringApp.Wpf.Modules.Alarm"
    xmlns:historyVm="clr-namespace:ModularMonitoringApp.Wpf.Modules.History"
    xmlns:historyView="clr-namespace:ModularMonitoringApp.Wpf.Modules.History"
    xmlns:configVm="clr-namespace:ModularMonitoringApp.Wpf.Modules.Configuration"
    xmlns:configView="clr-namespace:ModularMonitoringApp.Wpf.Modules.Configuration"
    Startup="Application_Startup"
    Exit="Application_Exit">

    <Application.Resources>

        <DataTemplate
            DataType="{x:Type deviceVm:DeviceViewModel}">
            <deviceView:DeviceView />
        </DataTemplate>

        <DataTemplate
            DataType="{x:Type monitorVm:MonitoringViewModel}">
            <monitorView:MonitoringView />
        </DataTemplate>

        <DataTemplate
            DataType="{x:Type alarmVm:AlarmViewModel}">
            <alarmView:AlarmView />
        </DataTemplate>

        <DataTemplate
            DataType="{x:Type historyVm:HistoryViewModel}">
            <historyView:HistoryView />
        </DataTemplate>

        <DataTemplate
            DataType="{x:Type configVm:ConfigurationViewModel}">
            <configView:ConfigurationView />
        </DataTemplate>

    </Application.Resources>

</Application>
```

The flow is:

```text
Shell.CurrentViewModel
        ↓
ContentControl
        ↓
WPF checks runtime type
        ↓
Find DataTemplate
        ↓
Create matching View
        ↓
View.DataContext = ViewModel
```

The navigation service therefore does not need to create Views directly.

---

# 39. Minimal Module ViewModels

These make the project complete while allowing the interesting modules to contain the performance logic.

## DeviceViewModel

```csharp
using ModularMonitoringApp.Core.Mvvm;

namespace ModularMonitoringApp.Wpf.Modules.Device;

public sealed class DeviceViewModel : ObservableObject
{
    public string Title => "Devices";
}
```

## ConfigurationViewModel

```csharp
using ModularMonitoringApp.Core.Mvvm;

namespace ModularMonitoringApp.Wpf.Modules.Configuration;

public sealed class ConfigurationViewModel : ObservableObject
{
    public string Title => "Configuration";
}
```

---

# 40. Minimal Views

## `DeviceView.xaml`

```xml
<UserControl
    x:Class="ModularMonitoringApp.Wpf.Modules.Device.DeviceView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <TextBlock
        FontSize="30"
        Text="{Binding Title}" />

</UserControl>
```

## `MonitoringView.xaml`

```xml
<UserControl
    x:Class="ModularMonitoringApp.Wpf.Modules.Monitoring.MonitoringView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <Grid>

        <DataGrid
            ItemsSource="{Binding Items}"
            EnableRowVirtualization="True"
            EnableColumnVirtualization="True"
            IsReadOnly="True" />

    </Grid>

</UserControl>
```

## `AlarmView.xaml`

```xml
<UserControl
    x:Class="ModularMonitoringApp.Wpf.Modules.Alarm.AlarmView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <DataGrid
        ItemsSource="{Binding Alarms}"
        IsReadOnly="True" />

</UserControl>
```

## `ConfigurationView.xaml`

```xml
<UserControl
    x:Class="ModularMonitoringApp.Wpf.Modules.Configuration.ConfigurationView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <TextBlock
        FontSize="30"
        Text="{Binding Title}" />

</UserControl>
```

Each XAML view requires a corresponding code-behind that only calls `InitializeComponent()`:

```csharp
public partial class DeviceView : UserControl
{
    public DeviceView()
    {
        InitializeComponent();
    }
}
```

The same pattern applies to MonitoringView, AlarmView, HistoryView, and ConfigurationView.

---

# 41. Dependency Injection

## WPF Project File

### `ModularMonitoringApp.Wpf.csproj`

```xml
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
        Include="Microsoft.Extensions.Hosting"
        Version="8.0.1" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference
        Include="..\ModularMonitoringApp.Core\ModularMonitoringApp.Core.csproj" />

    <ProjectReference
        Include="..\ModularMonitoringApp.Infrastructure\ModularMonitoringApp.Infrastructure.csproj" />
  </ItemGroup>

</Project>
```

---

# 42. Route Registration

### `Wpf/RouteRegistration.cs`

```csharp
using ModularMonitoringApp.Core.Navigation;
using ModularMonitoringApp.Wpf.Modules.Alarm;
using ModularMonitoringApp.Wpf.Modules.Configuration;
using ModularMonitoringApp.Wpf.Modules.Device;
using ModularMonitoringApp.Wpf.Modules.History;
using ModularMonitoringApp.Wpf.Modules.Monitoring;

namespace ModularMonitoringApp.Wpf;

public static class RouteRegistration
{
    public static void Register(
        IRouteRegistry routes)
    {
        routes.Register(
            "devices",
            typeof(DeviceViewModel));

        routes.Register(
            "monitoring",
            typeof(MonitoringViewModel));

        routes.Register(
            "alarms",
            typeof(AlarmViewModel));

        routes.Register(
            "history",
            typeof(HistoryViewModel));

        routes.Register(
            "configuration",
            typeof(ConfigurationViewModel));
    }
}
```

---

# 43. App Startup

## `App.xaml.cs`

```csharp
using System.Windows;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

using ModularMonitoringApp.Core.Contracts;
using ModularMonitoringApp.Core.Messaging;
using ModularMonitoringApp.Core.Navigation;

using ModularMonitoringApp.Infrastructure.Backend;
using ModularMonitoringApp.Infrastructure.Caching;
using ModularMonitoringApp.Infrastructure.Messaging;
using ModularMonitoringApp.Infrastructure.Streaming;

using ModularMonitoringApp.Wpf.Modules.Alarm;
using ModularMonitoringApp.Wpf.Modules.Configuration;
using ModularMonitoringApp.Wpf.Modules.Device;
using ModularMonitoringApp.Wpf.Modules.History;
using ModularMonitoringApp.Wpf.Modules.Monitoring;

namespace ModularMonitoringApp.Wpf;

public partial class App : Application
{
    private IHost? _host;
    private CancellationTokenSource? _appCts;

    private async void Application_Startup(
        object sender,
        StartupEventArgs e)
    {
        _appCts = new CancellationTokenSource();

        var builder =
            Host.CreateApplicationBuilder();

        ConfigureServices(builder.Services);

        _host = builder.Build();

        await _host.StartAsync(
            _appCts.Token);

        var routes =
            _host.Services.GetRequiredService<
                IRouteRegistry>();

        RouteRegistration.Register(routes);

        // Start infrastructure producer.
        var producer =
            _host.Services.GetRequiredService<
                TelemetryProducer>();

        _ = producer.RunAsync(_appCts.Token);

        var mainWindow =
            _host.Services.GetRequiredService<
                MainWindow>();

        mainWindow.Show();

        var navigation =
            _host.Services.GetRequiredService<
                INavigationService>();

        await navigation.NavigateAsync(
            "devices",
            _appCts.Token);
    }

    private async void Application_Exit(
        object sender,
        ExitEventArgs e)
    {
        _appCts?.Cancel();

        if (_host is not null)
        {
            await _host.StopAsync();

            _host.Dispose();
        }

        _appCts?.Dispose();
    }

    private static void ConfigureServices(
        IServiceCollection services)
    {
        // Navigation
        services.AddSingleton<IRouteRegistry,
            RouteRegistry>();

        services.AddSingleton<INavigationService,
            NavigationService>();

        // Shell
        services.AddSingleton<ShellViewModel>();

        services.AddSingleton<IShellNavigationHost>(
            provider =>
                provider.GetRequiredService<ShellViewModel>());

        services.AddSingleton<MainWindow>();

        // Infrastructure
        services.AddSingleton<TelemetryChannel>();

        services.AddSingleton<ITelemetryBackend,
            FakeTelemetryBackend>();

        services.AddSingleton<TelemetryProducer>();

        services.AddSingleton<IEventBus,
            EventBus>();

        services.AddSingleton<IHistoryBackend,
            FakeHistoryBackend>();

        services.AddSingleton<IPageCache<TelemetryData>,
            PageCache<TelemetryData>>();

        // ViewModels
        services.AddSingleton<DeviceViewModel>();

        services.AddSingleton<MonitoringViewModel>();

        services.AddSingleton<AlarmViewModel>();

        services.AddSingleton<HistoryViewModel>();

        services.AddSingleton<ConfigurationViewModel>();
    }
}
```

---

# 44. Lifetime Discussion

The example registers module ViewModels as singletons.

Why?

It makes the example simple and allows state to remain when navigating away.

But there is a trade-off.

```text
Singleton ViewModel
    ↓
State preserved
    ↓
May live for entire application
```

For large modules, this can waste memory.

An alternative is:

```text
Navigate
    ↓
Create scoped ViewModel
    ↓
Use
    ↓
Navigate away
    ↓
Dispose scope
```

A more advanced router could create a DI scope per navigation.

This is a good Staff-level discussion point.

---

# 45. Memory and Lifecycle Checklist

For each ViewModel that starts work:

```text
Does it have a CancellationTokenSource?
        ↓
Is it cancelled when inactive?
        ↓
Are subscriptions disposed?
        ↓
Are timers stopped?
        ↓
Are long-running Tasks observed?
        ↓
Are large collections bounded?
```

Common memory leak sources:

- event subscriptions
- static events
- timers
- long-running tasks
- retained ViewModels
- unbounded collections
- caches without eviction
- closures retaining Views/ViewModels

---

# 46. Testing Strategy

The main testing principle is:

> Business logic should not require a real WPF View.

Example test targets:

```text
ViewModel
Services
Navigation
Caching
Paging
Event handling
Cancellation
```

## Example: fake navigation backend

```csharp
public sealed class FakeHistoryBackend :
    IHistoryBackend
{
    public int CallCount { get; private set; }

    public Task<PageResult<TelemetryData>> GetPageAsync(
        PageRequest request,
        CancellationToken cancellationToken)
    {
        CallCount++;

        return Task.FromResult(
            new PageResult<TelemetryData>(
                new[]
                {
                    new TelemetryData(
                        "Device-1",
                        DateTime.UtcNow,
                        50,
                        20)
                },
                1));
    }
}
```

Test:

```csharp
[Fact]
public async Task LoadPage_WhenNotCached_CallsBackend()
{
    var backend = new FakeHistoryBackend();
    var cache = new PageCache<TelemetryData>();

    var viewModel =
        new HistoryViewModel(
            backend,
            cache);

    await viewModel.LoadPageAsync(
        0,
        CancellationToken.None);

    Assert.Equal(1, backend.CallCount);
    Assert.Single(viewModel.Items);
}
```

Second call:

```csharp
[Fact]
public async Task LoadPage_WhenCached_DoesNotCallBackendAgain()
{
    var backend = new FakeHistoryBackend();
    var cache = new PageCache<TelemetryData>();

    var viewModel =
        new HistoryViewModel(
            backend,
            cache);

    await viewModel.LoadPageAsync(
        0,
        CancellationToken.None);

    await viewModel.LoadPageAsync(
        0,
        CancellationToken.None);

    Assert.Equal(1, backend.CallCount);
}
```

---

# 47. Interview Scenario Walkthrough

A possible 60-minute interview can evolve like this.

## Question 1

> Design a modular WPF application.

Answer:

```text
Shell
    ↓
Navigation Service
    ↓
Route Registry
    ↓
DI
    ↓
ViewModel
    ↓
DataTemplate
    ↓
View
```

---

## Question 2

> We have 30 modules. Do we create everything at startup?

Answer:

> No. I would create the shell and core infrastructure first. Modules and their ViewModels should be activated when needed, depending on the required lifetime and initialization cost.

---

## Question 3

> The backend sends 2 GB of data.

Answer:

> First I would clarify whether this is historical data or a continuous stream. For historical data I would use paging or on-demand loading. For continuous data I would stream and process incrementally rather than loading everything into memory.

---

## Question 4

> The backend is faster than the UI.

Answer:

```text
Producer
    ↓
Bounded Channel
    ↓
Consumer
    ↓
Processing
    ↓
Batch
    ↓
Throttled UI updates
```

---

## Question 5

> The buffer becomes full.

Answer:

> I need to understand the data semantics. If this is a live dashboard, keeping the latest data may be more valuable than old data, so dropping old entries can be acceptable. If every message is business-critical, I would use backpressure or persistent storage rather than silently dropping messages.

---

## Question 6

> The user leaves the screen.

Answer:

> I would have an explicit navigation lifecycle. The ViewModel receives `OnNavigatedFromAsync`, cancels screen-specific work, stops timers, and removes event subscriptions. Whether the underlying backend stream stops depends on whether it is application-level or view-level infrastructure.

---

## Question 7

> How do modules communicate?

Answer:

```text
Request/Response
    ↓
Service Contract

Notification
    ↓
Event Bus

Shared application state
    ↓
Dedicated State Service
```

Avoid direct ViewModel-to-ViewModel dependencies.

---

## Question 8

> How do you prevent memory leaks?

Answer:

> I look at ownership and lifetime: event subscriptions, timers, cancellation tokens, long-running tasks, large collections, retained ViewModels, and caches. Every resource needs a clear owner and disposal/cancellation strategy.

---

# 48. Main Trade-offs

| Decision | Benefit | Cost |
|---|---|---|
| Singleton ViewModels | State preserved | Memory retained |
| Transient ViewModels | Fresh state | Reinitialization |
| Bounded Channel | Memory protection | Backpressure/data-loss policy needed |
| DropOldest | Latest UI data | Older data lost |
| Wait | No data loss | Producer slows |
| UI batching | Responsive UI | UI not updated per event |
| Paging | Low memory | More backend requests |
| Caching | Faster reads | Invalidation complexity |
| Event Bus | Loose coupling | Lifecycle/debugging complexity |
| Lifecycle callbacks | Controlled resources | More architecture |

---

# 49. The Most Important Architecture Statement

If you remember only one thing for the interview:

> **I separate data ingestion, business processing, application state, and UI rendering. I do not allow the UI collection to become the data ingestion pipeline. Each layer has an explicit lifetime, concurrency model, and backpressure strategy.**

That single statement naturally leads into:

```text
Streaming
Channel<T>
Buffering
Backpressure
Throttling
Batching
Caching
Paging
Cancellation
Navigation lifecycle
Virtualization
```

---

# 50. Recommended Next Improvements

This reference implementation is intentionally simple enough to understand. To make it more production-oriented, the next iteration should add:

1. `ObservableRangeCollection` or collection-reset batching.
2. A dedicated UI dispatcher abstraction so ViewModels are easier to test.
3. Structured logging and metrics.
4. Reconnection and retry policies for the telemetry backend.
5. Error state exposed to the UI.
6. LRU/TTL page caching.
7. Navigation parameters.
8. DI scopes per module/navigation instance.
9. Event bus asynchronous dispatch policy.
10. Unit and integration tests.
11. Performance counters:
   - incoming messages/sec
   - processed messages/sec
   - dropped messages
   - UI batch size
   - buffer depth
   - memory usage

These improvements create additional Staff Engineer interview discussion points.

---

# Final Mental Model

```text
                         USER
                           │
                           ▼
                         SHELL
                           │
                    Navigation / Router
                           │
                           ▼
                         MODULE
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 VIEWMODEL     LIFECYCLE
                    │             │
                    ▼             ▼
                 SERVICES     Cancellation
                    │
                    ▼
              APPLICATION STATE
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     Cache       Event Bus     Streaming
                                 │
                                 ▼
                           Channel / Buffer
                                 │
                                 ▼
                              Backend
```

The design principle is:

```text
Do not optimize the UI first.

First:
    Understand the data flow.

Then:
    Identify where data accumulates.

Then:
    Put explicit limits on memory.

Then:
    separate processing rate from UI update rate.

Then:
    define ownership and lifecycle.
```
