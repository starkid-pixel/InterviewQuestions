# Chapter 17 — Theme and Styling Design

## Design question

> How would you design light/dark themes for a large WPF application?

## Resource structure

```text
Themes/
 |
 +-- Light/
 |    +-- Colors.xaml
 |    +-- Brushes.xaml
 |    +-- Controls.xaml
 |
 +-- Dark/
      +-- Colors.xaml
      +-- Brushes.xaml
      +-- Controls.xaml
```

## Colors

```xml
<Color x:Key="PrimaryColor">#0067C5</Color>
<Color x:Key="BackgroundColor">#FFFFFF</Color>
```

## Brushes

```xml
<SolidColorBrush
    x:Key="PrimaryBrush"
    Color="{DynamicResource PrimaryColor}" />

<SolidColorBrush
    x:Key="BackgroundBrush"
    Color="{DynamicResource BackgroundColor}" />
```

## Control style

```xml
<Style TargetType="{x:Type Button}">
    <Setter Property="Background"
            Value="{DynamicResource PrimaryBrush}" />
</Style>
```

## Theme switching

A theme manager can replace the application's merged dictionaries:

```csharp
public sealed class ThemeService
{
    public void ApplyTheme(Uri themeUri)
    {
        var dictionaries =
            Application.Current.Resources.MergedDictionaries;

        dictionaries.Clear();
        dictionaries.Add(
            new ResourceDictionary
            {
                Source = themeUri
            });
    }
}
```

For a production system, preserve shared dictionaries and replace only theme-specific dictionaries.

## StaticResource vs DynamicResource

Use `DynamicResource` where runtime theme replacement is required.

Use `StaticResource` for stable resources.

## Design concern

Do not make every control carry dozens of hard-coded colors. Centralize semantic resources:
- Primary
- Background
- Surface
- Error
- Warning
- Success
- Text
- Border

This makes theme changes manageable.
