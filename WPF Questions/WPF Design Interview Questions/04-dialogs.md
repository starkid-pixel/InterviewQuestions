# Chapter 4 — Dialog Design

## Design question

> A ViewModel needs to show a confirmation dialog but must not reference MessageBox or Window. Design it.

## Architecture

```text
ViewModel
   |
   v
IDialogService
   |
   v
WpfDialogService
   |
   +--> MessageBox
   +--> Custom Window
```

## Contract

```csharp
public interface IDialogService
{
    Task<bool> ConfirmAsync(
        string title,
        string message);

    Task ShowErrorAsync(
        string title,
        string message);
}
```

## Implementation

```csharp
public sealed class WpfDialogService : IDialogService
{
    public Task<bool> ConfirmAsync(
        string title,
        string message)
    {
        var result = MessageBox.Show(
            message,
            title,
            MessageBoxButton.YesNo,
            MessageBoxImage.Question);

        return Task.FromResult(result == MessageBoxResult.Yes);
    }

    public Task ShowErrorAsync(
        string title,
        string message)
    {
        MessageBox.Show(
            message,
            title,
            MessageBoxButton.OK,
            MessageBoxImage.Error);

        return Task.CompletedTask;
    }
}
```

## ViewModel

```csharp
public sealed class CustomerViewModel
{
    private readonly IDialogService _dialogs;

    public CustomerViewModel(IDialogService dialogs)
    {
        _dialogs = dialogs;
    }

    public async Task DeleteAsync()
    {
        var confirmed = await _dialogs.ConfirmAsync(
            "Delete Customer",
            "Are you sure?");

        if (!confirmed)
            return;

        // Delete customer.
    }
}
```

## Testing

```csharp
public sealed class FakeDialogService : IDialogService
{
    public bool NextConfirmation { get; set; }

    public Task<bool> ConfirmAsync(
        string title,
        string message)
        => Task.FromResult(NextConfirmation);

    public Task ShowErrorAsync(
        string title,
        string message)
        => Task.CompletedTask;
}
```

The ViewModel can now be tested without opening a Window.

## Custom dialog design

For complex dialogs, use:

```text
IDialogService
    |
DialogViewModel
    |
DialogWindow
    |
DialogResult
```

Avoid putting business logic inside the dialog Window.
