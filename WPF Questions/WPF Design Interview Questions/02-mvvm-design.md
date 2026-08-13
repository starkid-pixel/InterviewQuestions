# Chapter 2 — MVVM Design

## Design question

> How should View, ViewModel, Model, services, and application logic communicate?

```text
View
 |
 | Binding / Command
 v
ViewModel
 |
 | interfaces
 v
Application Service
 |
 v
Domain / Infrastructure
```

## ViewModel example

```csharp
public sealed class CustomerViewModel : ObservableObject
{
    private readonly ICustomerService _service;

    private string _name = "";

    public CustomerViewModel(ICustomerService service)
    {
        _service = service;
        SaveCommand = new AsyncRelayCommand(SaveAsync, CanSave);
    }

    public string Name
    {
        get => _name;
        set
        {
            if (SetProperty(ref _name, value))
                SaveCommand.NotifyCanExecuteChanged();
        }
    }

    public IAsyncCommand SaveCommand { get; }

    private bool CanSave() => !string.IsNullOrWhiteSpace(Name);

    private async Task SaveAsync()
    {
        await _service.SaveAsync(Name);
    }
}
```

## ObservableObject

```csharp
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
}
```

## Async command

```csharp
public interface IAsyncCommand : ICommand
{
    Task ExecuteAsync();
    void NotifyCanExecuteChanged();
}

public sealed class AsyncRelayCommand : IAsyncCommand
{
    private readonly Func<Task> _execute;
    private readonly Func<bool>? _canExecute;
    private bool _running;

    public AsyncRelayCommand(
        Func<Task> execute,
        Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter)
        => !_running && (_canExecute?.Invoke() ?? true);

    public async void Execute(object? parameter)
    {
        if (!CanExecute(parameter))
            return;

        try
        {
            _running = true;
            NotifyCanExecuteChanged();
            await _execute();
        }
        finally
        {
            _running = false;
            NotifyCanExecuteChanged();
        }
    }

    public event EventHandler? CanExecuteChanged;

    public void NotifyCanExecuteChanged()
        => CanExecuteChanged?.Invoke(this, EventArgs.Empty);

    public Task ExecuteAsync() => _execute();
}
```

## Should ViewModel know about View?

Prefer no.

Bad:

```csharp
Window window = new CustomerWindow();
window.Show();
```

Better:

```csharp
await _navigation.NavigateAsync<CustomerDetailsViewModel>(
    new CustomerNavigationParameter(customerId));
```

## When code-behind is acceptable

MVVM does not mean zero code-behind. View-specific behavior such as focus, animations, visual-state handling, and control-specific behavior can reasonably remain in the View.

## Interview follow-up

> What if a ViewModel becomes 5,000 lines?

Split it by responsibility:
- child ViewModels
- application services
- validation services
- navigation
- dialogs
- state management
- feature-specific commands

The goal is cohesion, not merely fewer lines.
