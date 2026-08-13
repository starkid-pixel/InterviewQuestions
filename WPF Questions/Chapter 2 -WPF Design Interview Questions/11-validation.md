# Chapter 11 — Validation Design

## Design question

> How do you design validation for a large WPF application?

## Validation layers

```text
UI validation
    |
ViewModel validation
    |
Application validation
    |
Domain rules
    |
Server validation
```

## ViewModel with INotifyDataErrorInfo

```csharp
public abstract class ValidatableObject :
    INotifyPropertyChanged,
    INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = [];

    public bool HasErrors => _errors.Count > 0;

    public event PropertyChangedEventHandler? PropertyChanged;
    public event EventHandler<DataErrorsChangedEventArgs>?
        ErrorsChanged;

    public IEnumerable GetErrors(string? propertyName)
    {
        if (propertyName is null)
            return _errors.Values.SelectMany(x => x);

        return _errors.TryGetValue(
            propertyName,
            out var errors)
            ? errors
            : [];
    }

    protected void SetErrors(
        string property,
        IEnumerable<string> errors)
    {
        var list = errors.ToList();

        if (list.Count == 0)
            _errors.Remove(property);
        else
            _errors[property] = list;

        ErrorsChanged?.Invoke(
            this,
            new DataErrorsChangedEventArgs(property));

        PropertyChanged?.Invoke(
            this,
            new PropertyChangedEventArgs(nameof(HasErrors)));
    }
}
```

## ViewModel validation

```csharp
public void ValidateName(string name)
{
    var errors = new List<string>();

    if (string.IsNullOrWhiteSpace(name))
        errors.Add("Name is required.");

    if (name.Length > 100)
        errors.Add("Name cannot exceed 100 characters.");

    SetErrors(nameof(Name), errors);
}
```

## Important distinction

Client validation improves user experience.

It must not be treated as the final security/business boundary. The server/domain layer must validate authoritative business rules.

## Async validation

For uniqueness checks:

```csharp
public async Task ValidateCustomerNumberAsync(
    string number,
    CancellationToken token)
{
    var exists =
        await _service.ExistsAsync(number, token);

    SetErrors(
        nameof(CustomerNumber),
        exists
            ? ["Customer number already exists."]
            : []);
}
```

Debounce async validation to avoid making an API request for every keystroke.
